# Utilities and Support API

HoloDeck provides a rich set of utility modules for template rendering, file processing, search, chunking, context generation, tool initialization, logging, validation, and more.

______________________________________________________________________

## Template Engine

Jinja2-based template rendering with restricted filters and YAML validation against the `AgentConfig` schema. Used by `holodeck init` to scaffold projects from built-in templates.

## `TemplateRenderer()`

Renders Jinja2 templates and validates output against schemas.

Provides safe template rendering with:

- Restricted Jinja2 filters for security
- YAML validation against AgentConfig schema
- Clear error messages for debugging

Initialize the TemplateRenderer with a secure Jinja2 environment.

Source code in `src/holodeck/lib/template_engine.py`

```
def __init__(self) -> None:
    """Initialize the TemplateRenderer with a secure Jinja2 environment."""
    # Create Jinja2 environment with strict mode (undefined variables cause errors)
    # autoescape disabled for YAML templates (appropriate for config generation)
    self.env = Environment(
        undefined=StrictUndefined,
        trim_blocks=True,
        lstrip_blocks=True,
        autoescape=select_autoescape(
            enabled_extensions=("html", "xml"),
            default_for_string=False,
            default=False,
        ),
    )

    # Only allow safe filters
    self._setup_safe_filters()
```

### `get_available_templates()`

Get available templates with metadata.

Discovers templates from the templates/ directory and extracts metadata (name, display_name, description) from their manifest.yaml files.

Returns:

| Type                   | Description                                                     |
| ---------------------- | --------------------------------------------------------------- |
| `list[dict[str, str]]` | List of dicts with 'value', 'display_name', 'description' keys. |
| `list[dict[str, str]]` | Returns empty list if templates directory doesn't exist.        |

Source code in `src/holodeck/lib/template_engine.py`

```
@staticmethod
def get_available_templates() -> list[dict[str, str]]:
    """Get available templates with metadata.

    Discovers templates from the templates/ directory and extracts
    metadata (name, display_name, description) from their manifest.yaml files.

    Returns:
        List of dicts with 'value', 'display_name', 'description' keys.
        Returns empty list if templates directory doesn't exist.
    """
    templates: list[dict[str, str]] = []

    for template_dir in TemplateRenderer._discover_template_dirs():
        manifest_path = template_dir / "manifest.yaml"
        with open(manifest_path) as f:
            data = yaml.safe_load(f)
        if data and isinstance(data, dict):
            templates.append(
                {
                    "value": str(data.get("name", template_dir.name)),
                    "display_name": str(
                        data.get("display_name", template_dir.name)
                    ),
                    "description": str(data.get("description", "")),
                }
            )

    return templates
```

### `list_available_templates()`

List all available built-in templates.

Discovers templates from the templates/ directory structure.

Returns:

| Type        | Description                                                   |
| ----------- | ------------------------------------------------------------- |
| `list[str]` | List of template names (e.g., \['conversational', 'research', |
| `list[str]` | 'customer-support'\])                                         |

Source code in `src/holodeck/lib/template_engine.py`

```
@staticmethod
def list_available_templates() -> list[str]:
    """List all available built-in templates.

    Discovers templates from the templates/ directory structure.

    Returns:
        List of template names (e.g., ['conversational', 'research',
        'customer-support'])
    """
    return [d.name for d in TemplateRenderer._discover_template_dirs()]
```

### `render_and_validate(template_path, variables)`

Render a Jinja2 template and validate output (for YAML files).

Combines rendering and validation in a safe way: only returns rendered content if both rendering and validation succeed. This is the recommended way to process agent.yaml templates.

Parameters:

| Name            | Type             | Description                                     | Default    |
| --------------- | ---------------- | ----------------------------------------------- | ---------- |
| `template_path` | `str`            | Path to the Jinja2 template file                | *required* |
| `variables`     | `dict[str, Any]` | Dictionary of variables to pass to the template | *required* |

Returns:

| Type  | Description                                         |
| ----- | --------------------------------------------------- |
| `str` | Rendered and validated template content as a string |

Raises:

| Type              | Description         |
| ----------------- | ------------------- |
| `InitError`       | If rendering fails  |
| `ValidationError` | If validation fails |

Source code in `src/holodeck/lib/template_engine.py`

```
def render_and_validate(self, template_path: str, variables: dict[str, Any]) -> str:
    """Render a Jinja2 template and validate output (for YAML files).

    Combines rendering and validation in a safe way: only returns
    rendered content if both rendering and validation succeed.
    This is the recommended way to process agent.yaml templates.

    Args:
        template_path: Path to the Jinja2 template file
        variables: Dictionary of variables to pass to the template

    Returns:
        Rendered and validated template content as a string

    Raises:
        InitError: If rendering fails
        ValidationError: If validation fails
    """
    # Render template first
    rendered = self.render_template(template_path, variables)

    # Determine if this is agent.yaml specifically (not all YAML files)
    template_file = Path(template_path)
    is_agent_yaml = (
        template_file.name == "agent.yaml.j2" or template_file.stem == "agent.yaml"
    )

    if is_agent_yaml:
        # Validate YAML against schema
        # This will raise ValidationError if invalid
        self.validate_agent_config(rendered)

    # Return rendered content (safe to write to disk)
    return rendered
```

### `render_template(template_path, variables)`

Render a Jinja2 template with provided variables.

Parameters:

| Name            | Type             | Description                                     | Default    |
| --------------- | ---------------- | ----------------------------------------------- | ---------- |
| `template_path` | `str`            | Path to the Jinja2 template file                | *required* |
| `variables`     | `dict[str, Any]` | Dictionary of variables to pass to the template | *required* |

Returns:

| Type  | Description                           |
| ----- | ------------------------------------- |
| `str` | Rendered template content as a string |

Raises:

| Type                | Description                                                   |
| ------------------- | ------------------------------------------------------------- |
| `FileNotFoundError` | If template file doesn't exist                                |
| `InitError`         | If rendering fails (syntax errors, undefined variables, etc.) |

Source code in `src/holodeck/lib/template_engine.py`

```
def render_template(self, template_path: str, variables: dict[str, Any]) -> str:
    """Render a Jinja2 template with provided variables.

    Args:
        template_path: Path to the Jinja2 template file
        variables: Dictionary of variables to pass to the template

    Returns:
        Rendered template content as a string

    Raises:
        FileNotFoundError: If template file doesn't exist
        InitError: If rendering fails (syntax errors, undefined variables, etc.)
    """
    from holodeck.cli.exceptions import InitError

    template_file = Path(template_path)

    if not template_file.exists():
        raise FileNotFoundError(f"Template file not found: {template_path}")

    try:
        # Load template from file
        loader = FileSystemLoader(str(template_file.parent))
        env = Environment(
            loader=loader,
            undefined=StrictUndefined,
            trim_blocks=True,
            lstrip_blocks=True,
            autoescape=select_autoescape(
                enabled_extensions=("html", "xml"),
                default_for_string=False,
                default=False,
            ),
        )

        template = env.get_template(template_file.name)

        # Render template
        return template.render(variables)

    except TemplateSyntaxError as e:
        raise InitError(
            f"Template syntax error in {template_path}:\n"
            f"  Line {e.lineno}: {e.message}"
        ) from e
    except UndefinedError as e:
        raise InitError(
            f"Template rendering error in {template_path}:\n"
            f"  Undefined variable: {str(e)}"
        ) from e
    except Exception as e:
        raise InitError(f"Template rendering failed: {str(e)}") from e
```

### `validate_agent_config(yaml_content)`

Validate YAML content against Agent schema.

Parses YAML and validates it against the Agent Pydantic model. This is the critical validation gate for agent.yaml files.

Parameters:

| Name           | Type  | Description              | Default    |
| -------------- | ----- | ------------------------ | ---------- |
| `yaml_content` | `str` | YAML content as a string | *required* |

Returns:

| Name    | Type    | Description                          |
| ------- | ------- | ------------------------------------ |
| `Agent` | `Agent` | Validated Agent configuration object |

Raises:

| Type              | Description                                |
| ----------------- | ------------------------------------------ |
| `ValidationError` | If YAML is invalid or doesn't match schema |
| `InitError`       | If parsing fails                           |

Source code in `src/holodeck/lib/template_engine.py`

```
def validate_agent_config(self, yaml_content: str) -> Agent:
    """Validate YAML content against Agent schema.

    Parses YAML and validates it against the Agent Pydantic model.
    This is the critical validation gate for agent.yaml files.

    Args:
        yaml_content: YAML content as a string

    Returns:
        Agent: Validated Agent configuration object

    Raises:
        ValidationError: If YAML is invalid or doesn't match schema
        InitError: If parsing fails
    """
    from holodeck.cli.exceptions import InitError, ValidationError

    try:
        # Parse YAML
        data = yaml.safe_load(yaml_content)

        if not data:
            raise ValidationError("agent.yaml content is empty")

        # Validate against Agent schema
        agent = Agent.model_validate(data)
        return agent

    except yaml.YAMLError as e:
        raise ValidationError(f"YAML parsing error:\n" f"  {str(e)}") from e
    except ValidationError:
        # Re-raise our validation errors as-is
        raise
    except Exception as e:
        # Catch Pydantic validation errors
        if hasattr(e, "errors"):
            # Pydantic ValidationError
            errors = e.errors()
            error_msg = "Agent configuration validation failed:\n"
            for error in errors:
                field = ".".join(str(loc) for loc in error["loc"])
                error_msg += f"  {field}: {error['msg']}\n"
            raise ValidationError(error_msg) from e
        else:
            raise InitError(
                f"Agent configuration validation failed: {str(e)}"
            ) from e
```

______________________________________________________________________

## File Processing

Multimodal file processor that converts Office documents, PDFs, images (OCR), CSV, and JSON into markdown for LLM consumption. Supports local and remote files with caching, page/sheet/range extraction, and configurable timeouts.

## `SourceFile(path, content='', mtime=0.0, size_bytes=0, file_type='', chunks=list())`

Source file to be ingested into vector store.

Represents a file during the ingestion process with metadata and content.

Attributes:

| Name         | Type        | Description                                                     |
| ------------ | ----------- | --------------------------------------------------------------- |
| `path`       | `Path`      | Absolute file path                                              |
| `content`    | `str`       | File content converted to markdown (populated by FileProcessor) |
| `mtime`      | `float`     | File modification time (Unix timestamp)                         |
| `size_bytes` | `int`       | File size in bytes                                              |
| `file_type`  | `str`       | File extension (.txt, .md, .pdf, .csv, .json, etc.)             |
| `chunks`     | `list[str]` | Text chunks after splitting (populated by TextChunker)          |

## `FileProcessor(cache_dir=None, download_timeout_ms=30000, max_retries=3, processing_timeout_ms=30000)`

Process files with markitdown for multimodal test inputs.

Initialize file processor.

Parameters:

| Name                    | Type  | Description                                                       | Default                                                          |
| ----------------------- | ----- | ----------------------------------------------------------------- | ---------------------------------------------------------------- |
| `cache_dir`             | \`str | None\`                                                            | Directory for caching remote files. Defaults to .holodeck/cache/ |
| `download_timeout_ms`   | `int` | Timeout for file downloads in milliseconds                        | `30000`                                                          |
| `max_retries`           | `int` | Maximum number of retry attempts for downloads                    | `3`                                                              |
| `processing_timeout_ms` | `int` | Timeout for file processing in milliseconds. Defaults to 30000ms. | `30000`                                                          |

Source code in `src/holodeck/lib/file_processor.py`

```
def __init__(
    self,
    cache_dir: str | None = None,
    download_timeout_ms: int = 30000,
    max_retries: int = 3,
    processing_timeout_ms: int = 30000,
) -> None:
    """Initialize file processor.

    Args:
        cache_dir: Directory for caching remote files. Defaults to .holodeck/cache/
        download_timeout_ms: Timeout for file downloads in milliseconds
        max_retries: Maximum number of retry attempts for downloads
        processing_timeout_ms: Timeout for file processing in milliseconds.
            Defaults to 30000ms.
    """
    self.cache_dir = Path(cache_dir or ".holodeck/cache")
    self.cache_dir.mkdir(parents=True, exist_ok=True)

    self.download_timeout_ms = download_timeout_ms
    self.max_retries = max_retries
    self.processing_timeout_ms = processing_timeout_ms
    self.md: Any = None  # Initialize lazily

    logger.debug(
        f"FileProcessor initialized: cache_dir={self.cache_dir}, "
        f"download_timeout={download_timeout_ms}ms, "
        f"processing_timeout={processing_timeout_ms}ms, max_retries={max_retries}"
    )

    try:
        from markitdown import MarkItDown  # noqa: F401
    except ImportError as e:
        logger.error("markitdown package not found", exc_info=True)
        raise ImportError(
            "markitdown is required for file processing. "
            "Install with: pip install 'markitdown[all]'"
        ) from e
```

### `from_execution_config(config, cache_dir=None, max_retries=3)`

Create FileProcessor from ExecutionConfig.

Factory method that handles conversion from ExecutionConfig's seconds-based timeouts to FileProcessor's milliseconds-based timeouts.

Parameters:

| Name          | Type              | Description                                      | Default                                                 |
| ------------- | ----------------- | ------------------------------------------------ | ------------------------------------------------------- |
| `config`      | `ExecutionConfig` | ExecutionConfig with timeout settings in seconds | *required*                                              |
| `cache_dir`   | \`str             | None\`                                           | Override cache directory (defaults to config.cache_dir) |
| `max_retries` | `int`             | Maximum retry attempts for downloads             | `3`                                                     |

Returns:

| Type            | Description                       |
| --------------- | --------------------------------- |
| `FileProcessor` | Configured FileProcessor instance |

Source code in `src/holodeck/lib/file_processor.py`

```
@classmethod
def from_execution_config(
    cls,
    config: "ExecutionConfig",
    cache_dir: str | None = None,
    max_retries: int = 3,
) -> "FileProcessor":
    """Create FileProcessor from ExecutionConfig.

    Factory method that handles conversion from ExecutionConfig's
    seconds-based timeouts to FileProcessor's milliseconds-based timeouts.

    Args:
        config: ExecutionConfig with timeout settings in seconds
        cache_dir: Override cache directory (defaults to config.cache_dir)
        max_retries: Maximum retry attempts for downloads

    Returns:
        Configured FileProcessor instance
    """
    # Convert seconds to milliseconds, using defaults if not specified
    download_timeout_ms = (config.download_timeout or 30) * 1000
    processing_timeout_ms = (config.file_timeout or 30) * 1000

    return cls(
        cache_dir=cache_dir or config.cache_dir or ".holodeck/cache",
        download_timeout_ms=download_timeout_ms,
        processing_timeout_ms=processing_timeout_ms,
        max_retries=max_retries,
    )
```

### `process_file(file_input)`

Process a single file input to markdown.

Parameters:

| Name         | Type        | Description                               | Default    |
| ------------ | ----------- | ----------------------------------------- | ---------- |
| `file_input` | `FileInput` | File input configuration with path or URL | *required* |

Returns:

| Type                 | Description                                           |
| -------------------- | ----------------------------------------------------- |
| `ProcessedFileInput` | ProcessedFileInput with markdown content and metadata |

Source code in `src/holodeck/lib/file_processor.py`

```
def process_file(self, file_input: FileInput) -> ProcessedFileInput:
    """Process a single file input to markdown.

    Args:
        file_input: File input configuration with path or URL

    Returns:
        ProcessedFileInput with markdown content and metadata
    """
    start_time = time.time()
    file_location = file_input.url or file_input.path or "unknown"

    logger.debug(
        f"Processing file: {file_location} (type={file_input.type}, "
        f"cache={'enabled' if file_input.cache else 'disabled'})"
    )

    try:
        # Determine if file is local or remote
        if file_input.url:
            return self._process_remote_file(file_input, start_time)
        else:
            return self._process_local_file(file_input, start_time)

    except Exception as e:
        elapsed_ms = int((time.time() - start_time) * 1000)
        log_exception(logger, f"File processing failed: {file_location}", e)
        # Create detailed error message with context
        error_msg = self._format_error_message(e, file_location)
        return ProcessedFileInput(
            original=file_input,
            markdown_content="",
            metadata={},
            processing_time_ms=elapsed_ms,
            cached_path=None,
            error=error_msg,
        )
```

______________________________________________________________________

## Search

Hybrid search combining semantic (vector) similarity with keyword (full-text) matching via Reciprocal Rank Fusion (RRF). Includes tiered strategy selection based on vector store provider capabilities.

### Hybrid Search

## `SearchResult(chunk_id, content, fused_score, source_path, parent_chain, section_id, subsection_ids=list(), semantic_score=None, keyword_score=None, exact_match=False, definitions_context=list())`

A single result from hybrid search.

Represents a search result that may include scores from both semantic (vector) and keyword (full-text) search, along with document structure metadata and related definitions.

Attributes:

| Name                  | Type                    | Description                                               |
| --------------------- | ----------------------- | --------------------------------------------------------- |
| `chunk_id`            | `str`                   | Unique identifier of the matched chunk                    |
| `content`             | `str`                   | The text content of the matched chunk                     |
| `fused_score`         | `float`                 | Combined score from semantic and keyword search (0.0-1.0) |
| `source_path`         | `str`                   | Path to the source document file                          |
| `parent_chain`        | `list[str]`             | List of ancestor headings from root to immediate parent   |
| `section_id`          | `str`                   | Document section identifier (e.g., "1.2.3")               |
| `subsection_ids`      | `list[str]`             | List of inline subsection IDs contained in this chunk     |
| `semantic_score`      | \`float                 | None\`                                                    |
| `keyword_score`       | \`float                 | None\`                                                    |
| `exact_match`         | `bool`                  | Whether this result contains an exact phrase match        |
| `definitions_context` | `list[DefinitionEntry]` | Related definitions for terms found in the content        |

Example

> > > result = SearchResult( ... chunk_id="policy_md_chunk_5", ... content="Force Majeure means any event...", ... fused_score=0.92, ... source_path="/docs/policy.md", ... parent_chain=["Chapter 1", "Definitions"], ... section_id="1.2", ... subsection_ids=["subsec_a_findings", "para_1_access"], ... semantic_score=0.88, ... keyword_score=0.95, ... exact_match=True, ... ) print(result.format())

### `format()`

Format result for agent consumption.

Produces a human-readable representation of the search result suitable for inclusion in agent context or display to users.

Returns:

| Type  | Description                                             |
| ----- | ------------------------------------------------------- |
| `str` | Formatted string with score, source, location, content, |
| `str` | and any relevant definitions.                           |

Example

> > > result = SearchResult( ... chunk_id="doc_0", ... content="Hello world", ... fused_score=0.85, ... source_path="/doc.md", ... parent_chain=["Chapter 1"], ... section_id="1.1", ... ) print(result.format()) Score: 0.850 | Source: /doc.md Location: Chapter 1 Section: 1.1 Hello world

Source code in `src/holodeck/lib/hybrid_search.py`

```
def format(self) -> str:
    """Format result for agent consumption.

    Produces a human-readable representation of the search result
    suitable for inclusion in agent context or display to users.

    Returns:
        Formatted string with score, source, location, content,
        and any relevant definitions.

    Example:
        >>> result = SearchResult(
        ...     chunk_id="doc_0",
        ...     content="Hello world",
        ...     fused_score=0.85,
        ...     source_path="/doc.md",
        ...     parent_chain=["Chapter 1"],
        ...     section_id="1.1",
        ... )
        >>> print(result.format())
        Score: 0.850 | Source: /doc.md
        Location: Chapter 1
        Section: 1.1
        <BLANKLINE>
        Hello world
    """
    location = " > ".join(self.parent_chain) if self.parent_chain else "Root"
    lines = [
        f"Score: {self.fused_score:.3f} | Source: {self.source_path}",
        f"Location: {location}",
    ]
    if self.section_id:
        lines.append(f"Section: {self.section_id}")
    lines.append("")
    lines.append(self.content)
    if self.definitions_context:
        lines.append("")
        lines.append("Relevant definitions:")
        for defn in self.definitions_context:
            text = defn.definition_text
            if len(text) > 100:
                text = text[:97] + "..."
            lines.append(f"  - {defn.term}: {text}")
    return "\n".join(lines)
```

## `reciprocal_rank_fusion(ranked_lists, k=60, weights=None)`

Merge multiple ranked lists using Reciprocal Rank Fusion (RRF).

RRF combines results from different retrieval systems by scoring each document based on its rank in each list: score(d) = Σ weight_i / (k + rank_i(d))

This approach is robust to different score distributions across retrieval systems and doesn't require score calibration.

Parameters:

| Name           | Type                            | Description                                                                                                              | Default                                                                                                 |
| -------------- | ------------------------------- | ------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------- |
| `ranked_lists` | `list[list[tuple[str, float]]]` | List of ranked result lists, each containing (doc_id, score) tuples sorted by relevance descending.                      | *required*                                                                                              |
| `k`            | `int`                           | RRF constant (default 60). Higher values give more weight to lower-ranked results, reducing the impact of rank position. | `60`                                                                                                    |
| `weights`      | \`list[float]                   | None\`                                                                                                                   | Optional weights for each list (default equal weights). Use to prioritize certain retrieval modalities. |

Returns:

| Type                      | Description                                                         |
| ------------------------- | ------------------------------------------------------------------- |
| `list[tuple[str, float]]` | Merged list of (doc_id, score) tuples sorted by RRF score.          |
| `list[tuple[str, float]]` | Scores are normalized to 0-1 range based on maximum possible score. |

Example

> > > semantic = [("a", 0.9), ("b", 0.8), ("c", 0.7)] keyword = [("b", 0.95), ("a", 0.85), ("d", 0.75)] fused = reciprocal_rank_fusion([semantic, keyword], k=60) print(fused[0]) # Most relevant document ('b', 0.032...)

Source code in `src/holodeck/lib/hybrid_search.py`

```
def reciprocal_rank_fusion(
    ranked_lists: list[list[tuple[str, float]]],
    k: int = 60,
    weights: list[float] | None = None,
) -> list[tuple[str, float]]:
    """Merge multiple ranked lists using Reciprocal Rank Fusion (RRF).

    RRF combines results from different retrieval systems by scoring
    each document based on its rank in each list:
        score(d) = Σ weight_i / (k + rank_i(d))

    This approach is robust to different score distributions across
    retrieval systems and doesn't require score calibration.

    Args:
        ranked_lists: List of ranked result lists, each containing
            (doc_id, score) tuples sorted by relevance descending.
        k: RRF constant (default 60). Higher values give more weight
            to lower-ranked results, reducing the impact of rank position.
        weights: Optional weights for each list (default equal weights).
            Use to prioritize certain retrieval modalities.

    Returns:
        Merged list of (doc_id, score) tuples sorted by RRF score.
        Scores are normalized to 0-1 range based on maximum possible score.

    Example:
        >>> semantic = [("a", 0.9), ("b", 0.8), ("c", 0.7)]
        >>> keyword = [("b", 0.95), ("a", 0.85), ("d", 0.75)]
        >>> fused = reciprocal_rank_fusion([semantic, keyword], k=60)
        >>> print(fused[0])  # Most relevant document
        ('b', 0.032...)
    """
    if not ranked_lists:
        return []

    # Default to equal weights
    if weights is None:
        weights = [1.0] * len(ranked_lists)

    # Normalize weights
    total_weight = sum(weights)
    if total_weight > 0:
        weights = [w / total_weight for w in weights]

    # Calculate RRF scores
    scores: dict[str, float] = {}
    for weight, ranked_list in zip(weights, ranked_lists, strict=False):
        for rank, (doc_id, _) in enumerate(ranked_list, start=1):
            if doc_id not in scores:
                scores[doc_id] = 0.0
            scores[doc_id] += weight / (k + rank)

    # Sort by fused score descending
    sorted_results = sorted(scores.items(), key=lambda x: x[1], reverse=True)

    # Normalize scores to 0-1 range
    # Maximum possible score is when doc is rank 1 in all lists with max weights
    max_possible_score = sum(w / (k + 1) for w in weights) if weights else 1.0
    if max_possible_score > 0:
        normalized = [
            (doc_id, score / max_possible_score) for doc_id, score in sorted_results
        ]
        return normalized

    return sorted_results
```

### Keyword Search

## `KeywordSearchStrategy`

Bases: `str`, `Enum`

Keyword search strategy based on provider capabilities.

Determines whether to use native hybrid search or BM25 fallback:

- NATIVE_HYBRID: Provider supports hybrid_search() API directly
- FALLBACK_BM25: Use rank_bm25 + app-level RRF fusion

The strategy is automatically selected based on the vector store provider.

## `KeywordDocument`

Bases: `_KeywordDocumentRequired`

Structured document for multi-field keyword indexing.

Contains the content and metadata fields extracted from DocumentChunk that are relevant for keyword-based retrieval. Fields are used for multi-field indexing with per-field boosting.

Required Attributes

id: Unique chunk identifier. content: Primary text content (contextualized_content or content fallback).

Optional Attributes

parent_chain: Ancestor heading chain joined with " > " (e.g., "Chapter 1 > Definitions"). section_id: Document section identifier (e.g., "1.2.3", "203(a)"). defined_term: The term being defined (if chunk_type is definition). chunk_type: Classification of content type (content, definition, requirement, etc.). source_file: Source filename extracted from source_path.

## `KeywordSearchProvider`

Bases: `Protocol`

Protocol for keyword search backends.

Defines the interface that all keyword search providers must implement. Uses structural subtyping (Protocol) so providers satisfy the interface via duck typing without explicit inheritance.

Methods:

| Name     | Description                                         |
| -------- | --------------------------------------------------- |
| `build`  | Index documents for keyword search.                 |
| `search` | Search indexed documents and return ranked results. |

### `build(documents)`

Build keyword index from documents.

Parameters:

| Name        | Type                    | Description                                           | Default    |
| ----------- | ----------------------- | ----------------------------------------------------- | ---------- |
| `documents` | `list[KeywordDocument]` | List of KeywordDocument dicts with structured fields. | *required* |

Source code in `src/holodeck/lib/keyword_search.py`

```
def build(self, documents: list[KeywordDocument]) -> None:
    """Build keyword index from documents.

    Args:
        documents: List of KeywordDocument dicts with structured fields.
    """
    ...
```

### `search(query, top_k=10)`

Search indexed documents for matching results.

Parameters:

| Name    | Type  | Description                          | Default    |
| ------- | ----- | ------------------------------------ | ---------- |
| `query` | `str` | Search query string.                 | *required* |
| `top_k` | `int` | Maximum number of results to return. | `10`       |

Returns:

| Type                      | Description                                                |
| ------------------------- | ---------------------------------------------------------- |
| `list[tuple[str, float]]` | List of (doc_id, score) tuples sorted by score descending. |

Source code in `src/holodeck/lib/keyword_search.py`

```
def search(self, query: str, top_k: int = 10) -> list[tuple[str, float]]:
    """Search indexed documents for matching results.

    Args:
        query: Search query string.
        top_k: Maximum number of results to return.

    Returns:
        List of (doc_id, score) tuples sorted by score descending.
    """
    ...
```

## `InMemoryBM25KeywordProvider(k1=1.5, b=0.75)`

In-memory BM25 keyword search provider with multi-field indexing.

Provides keyword-based search using the BM25Okapi algorithm from the rank_bm25 library. Indexes multiple document fields (content, parent_chain, section_id, defined_term, source_file) with implicit boosting via field repetition.

Attributes:

| Name | Type | Description                                            |
| ---- | ---- | ------------------------------------------------------ |
| `k1` |      | BM25 term frequency saturation parameter (default 1.5) |
| `b`  |      | BM25 length normalization parameter (default 0.75)     |

Example

> > > provider = InMemoryBM25KeywordProvider() provider.build([ ... KeywordDocument(id="doc1", content="The quick brown fox"), ... KeywordDocument(id="doc2", content="The lazy dog"), ... ]) results = provider.search("brown fox", top_k=2) print(results[0]) # (doc_id, score) tuple ('doc1', 1.234)

Initialize the BM25 provider.

Parameters:

| Name | Type    | Description                                                                                        | Default |
| ---- | ------- | -------------------------------------------------------------------------------------------------- | ------- |
| `k1` | `float` | BM25 k1 parameter for term frequency saturation. Higher values give more weight to term frequency. | `1.5`   |
| `b`  | `float` | BM25 b parameter for document length normalization. Higher values penalize longer documents more.  | `0.75`  |

Source code in `src/holodeck/lib/keyword_search.py`

```
def __init__(self, k1: float = 1.5, b: float = 0.75) -> None:
    """Initialize the BM25 provider.

    Args:
        k1: BM25 k1 parameter for term frequency saturation.
            Higher values give more weight to term frequency.
        b: BM25 b parameter for document length normalization.
            Higher values penalize longer documents more.
    """
    self.k1 = k1
    self.b = b
    self._bm25: Any = None  # BM25Okapi instance
    self._doc_ids: list[str] = []
```

### `build(documents)`

Build BM25 index from structured keyword documents.

Builds a composite text from each KeywordDocument's fields with implicit boosting (defined_term 3x, parent_chain 2x, section_id 2x, content 1x, source_file 1x), then indexes via BM25Okapi.

Parameters:

| Name        | Type                    | Description                                           | Default    |
| ----------- | ----------------------- | ----------------------------------------------------- | ---------- |
| `documents` | `list[KeywordDocument]` | List of KeywordDocument dicts with structured fields. | *required* |

Raises:

| Type          | Description                    |
| ------------- | ------------------------------ |
| `ImportError` | If rank_bm25 is not installed. |

Example

> > > provider.build([ ... KeywordDocument( ... id="chunk1", ... content="Force Majeure means any event...", ... parent_chain="Chapter 1 > Definitions", ... section_id="1.2", ... defined_term="Force Majeure", ... ), ... ])

Source code in `src/holodeck/lib/keyword_search.py`

```
def build(self, documents: list[KeywordDocument]) -> None:
    """Build BM25 index from structured keyword documents.

    Builds a composite text from each KeywordDocument's fields with
    implicit boosting (defined_term 3x, parent_chain 2x, section_id 2x,
    content 1x, source_file 1x), then indexes via BM25Okapi.

    Args:
        documents: List of KeywordDocument dicts with structured fields.

    Raises:
        ImportError: If rank_bm25 is not installed.

    Example:
        >>> provider.build([
        ...     KeywordDocument(
        ...         id="chunk1",
        ...         content="Force Majeure means any event...",
        ...         parent_chain="Chapter 1 > Definitions",
        ...         section_id="1.2",
        ...         defined_term="Force Majeure",
        ...     ),
        ... ])
    """
    from rank_bm25 import BM25Okapi

    self._doc_ids = [doc["id"] for doc in documents]
    tokenized = [_tokenize(_build_bm25_document(doc)) for doc in documents]
    self._bm25 = BM25Okapi(tokenized, k1=self.k1, b=self.b)

    logger.debug(f"Built BM25 index with {len(documents)} documents")
```

### `search(query, top_k=10)`

Search indexed documents for matching results.

Parameters:

| Name    | Type  | Description                         | Default    |
| ------- | ----- | ----------------------------------- | ---------- |
| `query` | `str` | Search query string                 | *required* |
| `top_k` | `int` | Maximum number of results to return | `10`       |

Returns:

| Type                      | Description                                                     |
| ------------------------- | --------------------------------------------------------------- |
| `list[tuple[str, float]]` | List of (doc_id, score) tuples sorted by BM25 score descending. |
| `list[tuple[str, float]]` | Returns empty list if index not built.                          |

Example

> > > results = provider.search("brown fox", top_k=5) [('doc1', 1.234), ('doc3', 0.567)]

Source code in `src/holodeck/lib/keyword_search.py`

```
def search(self, query: str, top_k: int = 10) -> list[tuple[str, float]]:
    """Search indexed documents for matching results.

    Args:
        query: Search query string
        top_k: Maximum number of results to return

    Returns:
        List of (doc_id, score) tuples sorted by BM25 score descending.
        Returns empty list if index not built.

    Example:
        >>> results = provider.search("brown fox", top_k=5)
        [('doc1', 1.234), ('doc3', 0.567)]
    """
    if not self._bm25:
        return []

    with tracer.start_as_current_span(
        "keyword.search.in_memory_bm25",
        attributes={
            "keyword.query_tokens": len(_tokenize(query)),
            "keyword.index_size": len(self._doc_ids),
            "keyword.top_k": top_k,
        },
    ) as span:
        query_tokens = _tokenize(query)
        scores = self._bm25.get_scores(query_tokens)

        # Get top-k indices sorted by score descending
        top_indices = scores.argsort()[-top_k:][::-1]

        results = [
            (self._doc_ids[i], float(scores[i]))
            for i in top_indices
            if scores[i] > 0
        ]

        span.set_attribute("keyword.result_count", len(results))
        return results
```

## `OpenSearchKeywordProvider(endpoint, index_name, username=None, password=None, api_key=None, verify_certs=True, timeout_seconds=10)`

OpenSearch-backed keyword search provider with multi-field indexing.

Uses the opensearch-py low-level client to index and search documents via BM25 scoring on an external OpenSearch cluster. Implements the KeywordSearchProvider protocol.

Indexes multiple fields from KeywordDocument with per-field boosting:

- content: Primary text (standard analyzer)
- parent_chain: Heading hierarchy (standard analyzer, 2x boost)
- section_id: Section identifiers (simple analyzer, 2x boost)
- defined_term: Definition terms (standard analyzer, 3x boost)
- chunk_type: Content classification (keyword type, filterable)
- source_file: Source filename (simple analyzer)

Attributes:

| Name              | Type | Description                         |
| ----------------- | ---- | ----------------------------------- |
| `endpoint`        |      | OpenSearch endpoint URL.            |
| `index_name`      |      | Name of the OpenSearch index.       |
| `verify_certs`    |      | Whether to verify TLS certificates. |
| `timeout_seconds` |      | Connection timeout in seconds.      |

Example

> > > provider = OpenSearchKeywordProvider( ... endpoint="https://search.example.com:9200", ... index_name="my-index", ... username="admin", ... password="secret", ... ) provider.build([KeywordDocument(id="doc1", content="The quick brown fox")]) results = provider.search("brown fox", top_k=5) print(results[0]) # (chunk_id, score) tuple ('doc1', 3.456)

Initialize the OpenSearch keyword provider.

Parameters:

| Name              | Type   | Description                                         | Default                                                 |
| ----------------- | ------ | --------------------------------------------------- | ------------------------------------------------------- |
| `endpoint`        | `str`  | OpenSearch endpoint URL (e.g. "https://host:9200"). | *required*                                              |
| `index_name`      | `str`  | Name of the index to create/use.                    | *required*                                              |
| `username`        | \`str  | None\`                                              | Basic auth username (used with password).               |
| `password`        | \`str  | None\`                                              | Basic auth password (used with username).               |
| `api_key`         | \`str  | None\`                                              | API key for authentication (alternative to basic auth). |
| `verify_certs`    | `bool` | Whether to verify TLS certificates.                 | `True`                                                  |
| `timeout_seconds` | `int`  | Connection timeout in seconds.                      | `10`                                                    |

Source code in `src/holodeck/lib/keyword_search.py`

```
def __init__(
    self,
    endpoint: str,
    index_name: str,
    username: str | None = None,
    password: str | None = None,
    api_key: str | None = None,
    verify_certs: bool = True,
    timeout_seconds: int = 10,
) -> None:
    """Initialize the OpenSearch keyword provider.

    Args:
        endpoint: OpenSearch endpoint URL (e.g. "https://host:9200").
        index_name: Name of the index to create/use.
        username: Basic auth username (used with password).
        password: Basic auth password (used with username).
        api_key: API key for authentication (alternative to basic auth).
        verify_certs: Whether to verify TLS certificates.
        timeout_seconds: Connection timeout in seconds.
    """
    import opensearchpy

    self.endpoint = endpoint
    self.index_name = index_name
    self.verify_certs = verify_certs
    self.timeout_seconds = timeout_seconds

    client_kwargs: dict[str, Any] = {
        "hosts": [endpoint],
        "verify_certs": verify_certs,
        "timeout": timeout_seconds,
    }

    if not verify_certs:
        client_kwargs["ssl_show_warn"] = False

    if api_key:
        client_kwargs["headers"] = {
            "Authorization": f"ApiKey {api_key}",
        }
    elif username and password:
        client_kwargs["http_auth"] = (username, password)

    self._client: opensearchpy.OpenSearch = opensearchpy.OpenSearch(**client_kwargs)
```

### `build(documents)`

Build OpenSearch index from structured keyword documents.

Creates the index if it does not exist. If the index already exists, clears all existing documents before re-indexing. Uses bulk indexing with refresh=True for immediate searchability.

Each KeywordDocument's fields are indexed into separate OpenSearch fields with per-field boosting applied at query time via multi_match.

Parameters:

| Name        | Type                    | Description                                           | Default    |
| ----------- | ----------------------- | ----------------------------------------------------- | ---------- |
| `documents` | `list[KeywordDocument]` | List of KeywordDocument dicts with structured fields. | *required* |

Raises:

| Type                  | Description                       |
| --------------------- | --------------------------------- |
| `OpenSearchException` | On connection or indexing errors. |

Source code in `src/holodeck/lib/keyword_search.py`

```
def build(self, documents: list[KeywordDocument]) -> None:
    """Build OpenSearch index from structured keyword documents.

    Creates the index if it does not exist. If the index already
    exists, clears all existing documents before re-indexing.
    Uses bulk indexing with refresh=True for immediate searchability.

    Each KeywordDocument's fields are indexed into separate OpenSearch
    fields with per-field boosting applied at query time via multi_match.

    Args:
        documents: List of KeywordDocument dicts with structured fields.

    Raises:
        opensearchpy.OpenSearchException: On connection or indexing errors.
    """
    import opensearchpy.helpers as os_helpers

    with tracer.start_as_current_span(
        "opensearch.build",
        attributes={
            "opensearch.index": self.index_name,
            "opensearch.document_count": len(documents),
        },
    ) as span:
        if not self._client.indices.exists(index=self.index_name):
            self._client.indices.create(
                index=self.index_name,
                body=self._INDEX_MAPPING,
            )
            logger.debug(f"Created OpenSearch index '{self.index_name}'")
        else:
            self._client.delete_by_query(
                index=self.index_name,
                body={"query": {"match_all": {}}},
                refresh=True,
            )
            logger.debug(
                f"Cleared existing documents from index '{self.index_name}'"
            )

        actions = [
            {
                "_index": self.index_name,
                "_id": doc["id"],
                "_source": {
                    "chunk_id": doc["id"],
                    "content": doc.get("content", ""),
                    "parent_chain": doc.get("parent_chain", ""),
                    "section_id": doc.get("section_id", ""),
                    "defined_term": doc.get("defined_term", ""),
                    "chunk_type": doc.get("chunk_type", ""),
                    "source_file": doc.get("source_file", ""),
                },
            }
            for doc in documents
        ]

        success_count, _ = os_helpers.bulk(self._client, actions, refresh=True)

        span.set_attribute("opensearch.indexed_count", success_count)
        logger.debug(
            f"Indexed {success_count}/{len(documents)} documents "
            f"into '{self.index_name}'"
        )
```

### `search(query, top_k=10)`

Search indexed documents using multi-field BM25 scoring.

Uses a multi_match query across all indexed fields with per-field boost factors (defined_term^3, parent_chain^2, section_id^2, content, source_file).

Parameters:

| Name    | Type  | Description                          | Default    |
| ------- | ----- | ------------------------------------ | ---------- |
| `query` | `str` | Search query string.                 | *required* |
| `top_k` | `int` | Maximum number of results to return. | `10`       |

Returns:

| Type                      | Description                                                       |
| ------------------------- | ----------------------------------------------------------------- |
| `list[tuple[str, float]]` | List of (chunk_id, score) tuples sorted by BM25 score descending. |
| `list[tuple[str, float]]` | Returns empty list if the index does not exist.                   |

Raises:

| Type                  | Description           |
| --------------------- | --------------------- |
| `OpenSearchException` | On connection errors. |

Source code in `src/holodeck/lib/keyword_search.py`

```
def search(self, query: str, top_k: int = 10) -> list[tuple[str, float]]:
    """Search indexed documents using multi-field BM25 scoring.

    Uses a multi_match query across all indexed fields with per-field
    boost factors (defined_term^3, parent_chain^2, section_id^2,
    content, source_file).

    Args:
        query: Search query string.
        top_k: Maximum number of results to return.

    Returns:
        List of (chunk_id, score) tuples sorted by BM25 score descending.
        Returns empty list if the index does not exist.

    Raises:
        opensearchpy.OpenSearchException: On connection errors.
    """
    with tracer.start_as_current_span(
        "opensearch.search",
        attributes={
            "opensearch.index": self.index_name,
            "opensearch.query": query,
            "opensearch.top_k": top_k,
        },
    ) as span:
        if not self._client.indices.exists(index=self.index_name):
            span.set_attribute("opensearch.result_count", 0)
            return []

        response = self._client.search(
            index=self.index_name,
            body={
                "query": {
                    "multi_match": {
                        "query": query,
                        "fields": self._SEARCH_FIELDS,
                        "type": "best_fields",
                        "operator": "or",
                    }
                },
                "size": top_k,
            },
        )

        results: list[tuple[str, float]] = []
        for hit in response["hits"]["hits"]:
            chunk_id = hit["_source"]["chunk_id"]
            score = float(hit["_score"])
            results.append((chunk_id, score))

        span.set_attribute("opensearch.result_count", len(results))
        return results
```

## `HybridSearchExecutor(provider, collection, semantic_weight=0.5, keyword_weight=0.3, rrf_k=60, keyword_index_config=None, chunk_decoder=None)`

Executes hybrid search using appropriate strategy for provider.

Routes search requests to either native hybrid search or BM25 fallback based on the provider's capabilities.

Attributes:

| Name         | Type | Description                             |
| ------------ | ---- | --------------------------------------- |
| `provider`   |      | Vector store provider name              |
| `collection` |      | Semantic Kernel vector store collection |
| `strategy`   |      | Determined keyword search strategy      |

Example

> > > executor = HybridSearchExecutor("weaviate", collection) executor.build_keyword_index(documents) # Optional for fallback results = await executor.search(query, embedding, top_k=10)

Initialize the hybrid search executor.

Parameters:

| Name                   | Type                                            | Description                                             | Default                                                                                                                                                                                                                                                                                         |
| ---------------------- | ----------------------------------------------- | ------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`             | `str`                                           | Vector store provider name (determines strategy)        | *required*                                                                                                                                                                                                                                                                                      |
| `collection`           | `Any`                                           | Semantic Kernel vector store collection instance        | *required*                                                                                                                                                                                                                                                                                      |
| `semantic_weight`      | `float`                                         | Weight for semantic results in RRF fusion (default 0.5) | `0.5`                                                                                                                                                                                                                                                                                           |
| `keyword_weight`       | `float`                                         | Weight for keyword results in RRF fusion (default 0.3)  | `0.3`                                                                                                                                                                                                                                                                                           |
| `rrf_k`                | `int`                                           | RRF ranking constant (default 60)                       | `60`                                                                                                                                                                                                                                                                                            |
| `keyword_index_config` | \`KeywordIndexConfig                            | None\`                                                  | Keyword index backend configuration. If None, defaults to in-memory BM25.                                                                                                                                                                                                                       |
| `chunk_decoder`        | \`Callable\[\[dict[str, Any]\], DocumentChunk\] | None\`                                                  | Optional callback to reconstruct a DocumentChunk from a raw payload dict. Required for native-hybrid providers — search pulls payloads inline and the executor populates \_chunk_map so the caller can resolve ids via get_chunk without a second round-trip. Unused by the BM25 fallback path. |

Source code in `src/holodeck/lib/keyword_search.py`

```
def __init__(
    self,
    provider: str,
    collection: Any,
    semantic_weight: float = 0.5,
    keyword_weight: float = 0.3,
    rrf_k: int = 60,
    keyword_index_config: KeywordIndexConfig | None = None,
    chunk_decoder: Callable[[dict[str, Any]], DocumentChunk] | None = None,
) -> None:
    """Initialize the hybrid search executor.

    Args:
        provider: Vector store provider name (determines strategy)
        collection: Semantic Kernel vector store collection instance
        semantic_weight: Weight for semantic results in RRF fusion (default 0.5)
        keyword_weight: Weight for keyword results in RRF fusion (default 0.3)
        rrf_k: RRF ranking constant (default 60)
        keyword_index_config: Keyword index backend configuration.
            If None, defaults to in-memory BM25.
        chunk_decoder: Optional callback to reconstruct a
            ``DocumentChunk`` from a raw payload dict. Required for
            native-hybrid providers — search pulls payloads inline
            and the executor populates ``_chunk_map`` so the caller
            can resolve ids via ``get_chunk`` without a second
            round-trip. Unused by the BM25 fallback path.
    """
    self.provider = provider
    self.collection = collection
    self.strategy = get_keyword_search_strategy(provider)
    self.semantic_weight = semantic_weight
    self.keyword_weight = keyword_weight
    self.rrf_k = rrf_k
    self.keyword_index_config = keyword_index_config
    self._chunk_decoder = chunk_decoder
    self._keyword_index: (
        InMemoryBM25KeywordProvider | OpenSearchKeywordProvider | None
    ) = None
    self._chunk_map: dict[str, DocumentChunk] = {}

    logger.debug(
        f"HybridSearchExecutor initialized: provider={provider}, "
        f"strategy={self.strategy.value}, "
        f"weights=semantic:{semantic_weight}/keyword:{keyword_weight}, "
        f"rrf_k={rrf_k}"
    )
```

### `build_keyword_index(chunks)`

Build keyword index with graceful degradation.

Creates the keyword search index from the provided chunks and stores a chunk map for ID-based lookups. Routes to the appropriate backend based on keyword_index_config:

- 'opensearch': OpenSearchKeywordProvider (I/O offloaded via asyncio.to_thread)
- 'in-memory' or None: InMemoryBM25KeywordProvider (called directly)

Each chunk is converted to a KeywordDocument with multiple fields (content, parent_chain, section_id, defined_term, source_file) for multi-field indexing with per-field boosting.

If index build fails, logs a warning and continues with semantic-only search.

Parameters:

| Name     | Type                  | Description                                                                                                                                | Default    |
| -------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------- |
| `chunks` | `list[DocumentChunk]` | List of DocumentChunk objects. All relevant fields (content, parent_chain, section_id, defined_term, etc.) are indexed for keyword search. | *required* |

Example

> > > await executor.build_keyword_index(chunks)

Source code in `src/holodeck/lib/keyword_search.py`

```
async def build_keyword_index(self, chunks: list[DocumentChunk]) -> None:
    """Build keyword index with graceful degradation.

    Creates the keyword search index from the provided chunks and stores
    a chunk map for ID-based lookups. Routes to the appropriate backend
    based on keyword_index_config:
    - 'opensearch': OpenSearchKeywordProvider (I/O offloaded via asyncio.to_thread)
    - 'in-memory' or None: InMemoryBM25KeywordProvider (called directly)

    Each chunk is converted to a KeywordDocument with multiple fields
    (content, parent_chain, section_id, defined_term, source_file) for
    multi-field indexing with per-field boosting.

    If index build fails, logs a warning and continues with
    semantic-only search.

    Args:
        chunks: List of DocumentChunk objects. All relevant fields
            (content, parent_chain, section_id, defined_term, etc.)
            are indexed for keyword search.

    Example:
        >>> await executor.build_keyword_index(chunks)
    """
    # Store chunk map for ID-based lookups
    self._chunk_map = {c.id: c for c in chunks}

    # Native-hybrid providers (qdrant) handle the keyword leg inside the
    # vector store at query time (see _native_hybrid_search_*). The
    # in-process BM25/OpenSearch index would never be read, so skip the
    # tokenize-and-build pass entirely. _chunk_map is still useful as a
    # cache for get_chunks() lookups; it just doesn't need to be the
    # source of truth for the keyword index.
    if self.strategy == KeywordSearchStrategy.NATIVE_HYBRID:
        logger.debug(
            "Skipping in-process keyword index build for native-hybrid "
            f"provider '{self.provider}' "
            f"(strategy={self.strategy.value})"
        )
        return

    # Convert chunks to structured keyword documents
    docs = [_chunk_to_keyword_document(c) for c in chunks]

    config = self.keyword_index_config
    provider_name = config.provider if config is not None else "in-memory"

    # Route to appropriate keyword search backend with OTel instrumentation
    with tracer.start_as_current_span(
        "keyword_index.build",
        attributes={
            "keyword_index.provider": provider_name,
            "keyword_index.document_count": len(docs),
        },
    ) as span:
        try:
            provider_inst: InMemoryBM25KeywordProvider | OpenSearchKeywordProvider
            if config is not None and config.provider == "opensearch":
                if not config.endpoint or not config.index_name:
                    raise ValueError(
                        "endpoint and index_name are required for "
                        "opensearch keyword index provider"
                    )
                provider_inst = OpenSearchKeywordProvider(
                    endpoint=config.endpoint,
                    index_name=config.index_name,
                    username=config.username,
                    password=(
                        config.password.get_secret_value()
                        if config.password is not None
                        else None
                    ),
                    api_key=(
                        config.api_key.get_secret_value()
                        if config.api_key is not None
                        else None
                    ),
                    verify_certs=config.verify_certs,
                    timeout_seconds=config.timeout_seconds,
                )
            else:
                provider_inst = InMemoryBM25KeywordProvider(k1=1.5, b=0.75)

            # Offload OpenSearch I/O to thread; call in-memory directly
            if isinstance(provider_inst, OpenSearchKeywordProvider):
                await asyncio.to_thread(provider_inst.build, docs)
            else:
                provider_inst.build(docs)
            self._keyword_index = provider_inst
            span.set_attribute("keyword_index.status", "success")
            logger.debug(f"Built keyword index with {len(chunks)} documents")
        except Exception as e:
            span.set_attribute("keyword_index.status", "failed")
            span.record_exception(e)
            logger.warning(
                f"Keyword index build failed, falling back to semantic-only: {e}"
            )
            self._keyword_index = None
```

### `get_chunk(chunk_id)`

Look up a chunk by ID from the stored chunk map.

Parameters:

| Name       | Type  | Description                         | Default    |
| ---------- | ----- | ----------------------------------- | ---------- |
| `chunk_id` | `str` | The unique identifier of the chunk. | *required* |

Returns:

| Type            | Description |
| --------------- | ----------- |
| \`DocumentChunk | None\`      |

Source code in `src/holodeck/lib/keyword_search.py`

```
def get_chunk(self, chunk_id: str) -> DocumentChunk | None:
    """Look up a chunk by ID from the stored chunk map.

    Args:
        chunk_id: The unique identifier of the chunk.

    Returns:
        The DocumentChunk if found, or None.
    """
    return self._chunk_map.get(chunk_id)
```

### `keyword_search(query, top_k=10)`

Perform keyword-only search.

Parameters:

| Name    | Type  | Description                          | Default    |
| ------- | ----- | ------------------------------------ | ---------- |
| `query` | `str` | Search query string.                 | *required* |
| `top_k` | `int` | Maximum number of results to return. | `10`       |

Returns:

| Type                      | Description                                                          |
| ------------------------- | -------------------------------------------------------------------- |
| `list[tuple[str, float]]` | List of (chunk_id, score) tuples sorted by score descending.         |
| `list[tuple[str, float]]` | Returns empty list if keyword index is not built or on search error. |

Source code in `src/holodeck/lib/keyword_search.py`

```
async def keyword_search(
    self, query: str, top_k: int = 10
) -> list[tuple[str, float]]:
    """Perform keyword-only search.

    Args:
        query: Search query string.
        top_k: Maximum number of results to return.

    Returns:
        List of (chunk_id, score) tuples sorted by score descending.
        Returns empty list if keyword index is not built or on search error.
    """
    if self._keyword_index is None:
        return []
    try:
        # Offload OpenSearch I/O to thread; call in-memory directly
        if isinstance(self._keyword_index, OpenSearchKeywordProvider):
            return await asyncio.to_thread(self._keyword_index.search, query, top_k)
        return self._keyword_index.search(query, top_k)
    except Exception as e:
        logger.warning(f"Keyword search failed, returning empty results: {e}")
        return []
```

### `search(query, query_embedding, top_k=10)`

Execute hybrid search and return ranked results.

Routes to native hybrid search or BM25 fallback based on provider.

Parameters:

| Name              | Type          | Description                         | Default    |
| ----------------- | ------------- | ----------------------------------- | ---------- |
| `query`           | `str`         | Search query string                 | *required* |
| `query_embedding` | `list[float]` | Pre-computed query embedding vector | *required* |
| `top_k`           | `int`         | Maximum number of results to return | `10`       |

Returns:

| Type                      | Description                                           |
| ------------------------- | ----------------------------------------------------- |
| `list[tuple[str, float]]` | List of (chunk_id, score) tuples sorted by relevance. |
| `list[tuple[str, float]]` | Scores are normalized to 0-1 range.                   |

Example

> > > results = await executor.search( ... "reporting requirements", ... [0.1, 0.2, ...], ... top_k=10 ... ) [('chunk_report', 0.95), ('chunk_other', 0.72), ...]

Source code in `src/holodeck/lib/keyword_search.py`

```
async def search(
    self,
    query: str,
    query_embedding: list[float],
    top_k: int = 10,
) -> list[tuple[str, float]]:
    """Execute hybrid search and return ranked results.

    Routes to native hybrid search or BM25 fallback based on provider.

    Args:
        query: Search query string
        query_embedding: Pre-computed query embedding vector
        top_k: Maximum number of results to return

    Returns:
        List of (chunk_id, score) tuples sorted by relevance.
        Scores are normalized to 0-1 range.

    Example:
        >>> results = await executor.search(
        ...     "reporting requirements",
        ...     [0.1, 0.2, ...],
        ...     top_k=10
        ... )
        [('chunk_report', 0.95), ('chunk_other', 0.72), ...]
    """
    with tracer.start_as_current_span(
        "hybrid_search.execute",
        attributes={
            "search.mode": self.strategy.value,
            "search.provider": self.provider,
            "search.top_k": top_k,
        },
    ) as span:
        # Execute hybrid search based on strategy
        if self.strategy == KeywordSearchStrategy.NATIVE_HYBRID:
            results = await self._native_hybrid_search(
                query, query_embedding, top_k
            )
        else:
            results = await self._fallback_hybrid_search(
                query, query_embedding, top_k
            )

        span.set_attribute("search.result_count", len(results))
        return results[:top_k]
```

## `get_keyword_search_strategy(provider)`

Determine keyword search strategy based on provider capabilities.

Parameters:

| Name       | Type  | Description                                                      | Default    |
| ---------- | ----- | ---------------------------------------------------------------- | ---------- |
| `provider` | `str` | Vector store provider name (e.g., "azure-ai-search", "postgres") | *required* |

Returns:

| Type                    | Description                                                     |
| ----------------------- | --------------------------------------------------------------- |
| `KeywordSearchStrategy` | KeywordSearchStrategy indicating native hybrid or BM25 fallback |

Example

> > > get_keyword_search_strategy("weaviate") KeywordSearchStrategy.NATIVE_HYBRID get_keyword_search_strategy("postgres") KeywordSearchStrategy.FALLBACK_BM25

Source code in `src/holodeck/lib/keyword_search.py`

```
def get_keyword_search_strategy(provider: str) -> KeywordSearchStrategy:
    """Determine keyword search strategy based on provider capabilities.

    Args:
        provider: Vector store provider name (e.g., "azure-ai-search", "postgres")

    Returns:
        KeywordSearchStrategy indicating native hybrid or BM25 fallback

    Example:
        >>> get_keyword_search_strategy("weaviate")
        KeywordSearchStrategy.NATIVE_HYBRID
        >>> get_keyword_search_strategy("postgres")
        KeywordSearchStrategy.FALLBACK_BM25
    """
    if provider in NATIVE_HYBRID_PROVIDERS:
        return KeywordSearchStrategy.NATIVE_HYBRID
    return KeywordSearchStrategy.FALLBACK_BM25
```

______________________________________________________________________

## Chunking

Text splitting and structure-aware chunking for preparing documents for embedding generation and hierarchical retrieval.

### Text Chunker

Token-based text splitting using Semantic Kernel's paragraph splitter.

## `TextChunker(chunk_size=DEFAULT_CHUNK_SIZE, chunk_overlap=DEFAULT_CHUNK_OVERLAP, separator_list=None)`

Wrapper for text chunking using Semantic Kernel.

Splits text into chunks of approximately equal size with token-based sizing. Uses Semantic Kernel's split functions for consistent chunk boundaries.

Attributes:

| Name            | Type | Description                                                       |
| --------------- | ---- | ----------------------------------------------------------------- |
| `chunk_size`    |      | Target number of tokens per chunk                                 |
| `chunk_overlap` |      | Overlapping tokens (note: not fully supported by Semantic Kernel) |

Initialize text chunker.

Parameters:

| Name             | Type        | Description                                                                                                                                                      | Default                                                     |
| ---------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------- |
| `chunk_size`     | `int`       | Target number of tokens per chunk (default: 512)                                                                                                                 | `DEFAULT_CHUNK_SIZE`                                        |
| `chunk_overlap`  | `int`       | Number of overlapping tokens between chunks (default: 50) Note: Semantic Kernel's split functions don't support overlap, so this is stored for API compatibility | `DEFAULT_CHUNK_OVERLAP`                                     |
| `separator_list` | \`list[str] | None\`                                                                                                                                                           | Custom list of separators (ignored, kept for compatibility) |

Raises:

| Type         | Description                              |
| ------------ | ---------------------------------------- |
| `ValueError` | If chunk_size \<= 0 or chunk_overlap < 0 |

Source code in `src/holodeck/lib/text_chunker.py`

```
def __init__(
    self,
    chunk_size: int = DEFAULT_CHUNK_SIZE,
    chunk_overlap: int = DEFAULT_CHUNK_OVERLAP,
    separator_list: list[str] | None = None,
) -> None:
    """Initialize text chunker.

    Args:
        chunk_size: Target number of tokens per chunk (default: 512)
        chunk_overlap: Number of overlapping tokens between chunks (default: 50)
            Note: Semantic Kernel's split functions don't support overlap,
            so this is stored for API compatibility
        separator_list: Custom list of separators (ignored, kept for compatibility)

    Raises:
        ValueError: If chunk_size <= 0 or chunk_overlap < 0
    """
    if chunk_size <= 0:
        raise ValueError("chunk_size must be positive")
    if chunk_overlap < 0:
        raise ValueError("chunk_overlap must be non-negative")

    self.chunk_size = chunk_size
    self.chunk_overlap = chunk_overlap
    self.separator_list = separator_list or ["\n\n", "\n", " ", ""]
```

### `split_text(text)`

Split text into chunks using Semantic Kernel's split functions.

Uses split_plaintext_paragraph to chunk text at paragraph boundaries when possible, falling back to line-based splitting for consistency.

Parameters:

| Name   | Type  | Description               | Default    |
| ------ | ----- | ------------------------- | ---------- |
| `text` | `str` | Text to split into chunks | *required* |

Returns:

| Type        | Description                                               |
| ----------- | --------------------------------------------------------- |
| `list[str]` | List of text chunks, each approximately chunk_size tokens |

Raises:

| Type           | Description                 |
| -------------- | --------------------------- |
| `ValueError`   | If text is empty            |
| `RuntimeError` | If chunking operation fails |

Source code in `src/holodeck/lib/text_chunker.py`

```
def split_text(self, text: str) -> list[str]:
    """Split text into chunks using Semantic Kernel's split functions.

    Uses split_plaintext_paragraph to chunk text at paragraph boundaries
    when possible, falling back to line-based splitting for consistency.

    Args:
        text: Text to split into chunks

    Returns:
        List of text chunks, each approximately chunk_size tokens

    Raises:
        ValueError: If text is empty
        RuntimeError: If chunking operation fails
    """
    if not text or not text.strip():
        raise ValueError("Text cannot be empty")

    try:
        # Split text into lines first
        lines = text.split("\n")

        # Use Semantic Kernel's paragraph splitter
        # It handles token counting automatically
        chunks_result = split_plaintext_paragraph(lines, max_tokens=self.chunk_size)

        # Filter out empty chunks
        chunks: list[str] = [chunk for chunk in chunks_result if chunk.strip()]
        return chunks
    except Exception as e:
        logger.error(f"Error during text chunking: {e}")
        raise RuntimeError(f"Failed to chunk text: {e}") from e
```

### Structured Chunker

Structure-aware markdown parsing with heading hierarchy extraction, chunk type classification, and parent chain building.

## `SubsectionPattern(name, pattern, level, extract_title=True)`

Configurable pattern for implicit subsection detection in documents.

Used to recognize legislative-style numbering schemes (e.g., (a), (1), (A)) as implicit headings that should create separate chunks with proper hierarchy.

Attributes:

| Name            | Type           | Description                                                                                                                                                              |
| --------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `name`          | `str`          | Human-readable name for the pattern (e.g., "subsection", "paragraph")                                                                                                    |
| `pattern`       | `Pattern[str]` | Compiled regex to match the marker. Should have groups for: - Group 1: The marker itself (e.g., "a", "1", "A") - Group 2: Optional text after the marker (title/content) |
| `level`         | `int`          | Heading level to assign (3-6, since 1-2 are reserved for markdown)                                                                                                       |
| `extract_title` | `bool`         | Whether to include text after marker as the heading title                                                                                                                |

Example

> > > pattern = SubsectionPattern( ... name="subsection", ... pattern=re.compile(r"^(([a-z]))\\s\*(.\*)$", re.MULTILINE), ... level=3, ... extract_title=True, ... )

## `ChunkType`

Bases: `str`, `Enum`

Classification of chunk content type.

Used to categorize document chunks for specialized handling during search and retrieval operations.

Attributes:

| Name          | Type | Description                                              |
| ------------- | ---- | -------------------------------------------------------- |
| `CONTENT`     |      | Regular document content (paragraphs, lists, etc.)       |
| `DEFINITION`  |      | Term definitions (e.g., glossary entries, defined terms) |
| `REQUIREMENT` |      | Requirements or obligations (e.g., "shall", "must")      |
| `REFERENCE`   |      | Cross-references or citations to other sections          |
| `HEADER`      |      | Section headers or titles                                |

## `DocumentChunk(id, source_path, chunk_index, content, parent_chain=list(), section_id='', chunk_type=ChunkType.CONTENT, cross_references=list(), heading_level=0, embedding=None, contextualized_content='', mtime=0.0, defined_term='', defined_term_normalized='', subsection_ids=list())`

A parsed section of a document with structure metadata.

Represents a single chunk of a document that preserves its position in the document hierarchy, cross-references to other sections, and optional definition information.

Attributes:

| Name                      | Type          | Description                                                            |
| ------------------------- | ------------- | ---------------------------------------------------------------------- |
| `id`                      | `str`         | Unique identifier for this chunk (typically source_path + chunk_index) |
| `source_path`             | `str`         | Path to the source document file                                       |
| `chunk_index`             | `int`         | Zero-based index of this chunk within the document                     |
| `content`                 | `str`         | The actual text content of the chunk                                   |
| `parent_chain`            | `list[str]`   | List of ancestor heading texts from root to immediate parent           |
| `section_id`              | `str`         | Document section identifier (e.g., "1.2.3", "A.1")                     |
| `chunk_type`              | `ChunkType`   | Classification of the content type                                     |
| `cross_references`        | `list[str]`   | List of section IDs referenced by this chunk                           |
| `heading_level`           | `int`         | Heading level if this is a header chunk (1-6, 0 for non-headers)       |
| `embedding`               | \`list[float] | None\`                                                                 |
| `contextualized_content`  | `str`         | Content with added context for better retrieval                        |
| `mtime`                   | `float`       | File modification time (Unix timestamp) for change detection           |
| `defined_term`            | `str`         | The term being defined (if chunk_type is DEFINITION)                   |
| `defined_term_normalized` | `str`         | Lowercase normalized term for case-insensitive lookup                  |

Example

> > > chunk = DocumentChunk( ... id="policy_md_chunk_5", ... source_path="/docs/policy.md", ... chunk_index=5, ... content="Force Majeure means any event beyond...", ... parent_chain=["Chapter 1", "Definitions"], ... section_id="1.2", ... chunk_type=ChunkType.DEFINITION, ... defined_term="Force Majeure", ... defined_term_normalized="force majeure", ... )

### `to_record_dict()`

Convert to dict for vector store record creation.

Serializes list fields as JSON strings for storage in vector databases that expect flat field structures.

Returns:

| Type             | Description                                                       |
| ---------------- | ----------------------------------------------------------------- |
| `dict[str, Any]` | Dictionary with all fields serialized for vector store insertion. |
| `dict[str, Any]` | List fields (parent_chain, cross_references) are JSON-encoded.    |

Example

> > > chunk = DocumentChunk( ... id="doc_0", ... source_path="/doc.md", ... chunk_index=0, ... content="Hello", ... parent_chain=["Ch1", "Sec1"], ... ) record = chunk.to_record_dict() record["parent_chain"] '["Ch1", "Sec1"]'

Source code in `src/holodeck/lib/structured_chunker.py`

```
def to_record_dict(self) -> dict[str, Any]:
    """Convert to dict for vector store record creation.

    Serializes list fields as JSON strings for storage in vector databases
    that expect flat field structures.

    Returns:
        Dictionary with all fields serialized for vector store insertion.
        List fields (parent_chain, cross_references) are JSON-encoded.

    Example:
        >>> chunk = DocumentChunk(
        ...     id="doc_0",
        ...     source_path="/doc.md",
        ...     chunk_index=0,
        ...     content="Hello",
        ...     parent_chain=["Ch1", "Sec1"],
        ... )
        >>> record = chunk.to_record_dict()
        >>> record["parent_chain"]
        '["Ch1", "Sec1"]'
    """
    import json

    return {
        "id": self.id,
        "source_path": self.source_path,
        "chunk_index": self.chunk_index,
        "content": self.content,
        "embedding": self.embedding,
        "parent_chain": json.dumps(self.parent_chain),
        "section_id": self.section_id,
        "chunk_type": self.chunk_type.value,
        "cross_references": json.dumps(self.cross_references),
        "contextualized_content": self.contextualized_content,
        "mtime": self.mtime,
        "defined_term": self.defined_term,
        "defined_term_normalized": self.defined_term_normalized,
        "subsection_ids": json.dumps(self.subsection_ids),
    }
```

## `StructuredChunker(max_tokens=DEFAULT_MAX_TOKENS, subsection_patterns=None, max_subsection_depth=None, split_on_level=None)`

Structure-aware markdown chunker with hierarchy preservation.

Parses markdown documents into chunks while preserving:

- Parent chain (heading hierarchy for navigation context)
- Section IDs (normalized identifiers)
- Chunk type classification (content, definition, requirement, etc.)
- Token-bounded sections with sentence-aware splitting

The chunker follows Anthropic's contextual retrieval baseline with a default max_tokens of 800 per chunk.

Attributes:

| Name         | Type  | Description                             |
| ------------ | ----- | --------------------------------------- |
| `max_tokens` | `int` | Maximum tokens per chunk (default 800). |

Example

> > > chunker = StructuredChunker(max_tokens=800) chunks = chunker.parse(markdown_content, "document.md") for chunk in chunks: ... print(f"{chunk.section_id}: {chunk.parent_chain}")

Initialize the structured chunker.

Parameters:

| Name                   | Type                      | Description                                | Default                                                                                                                                                                                                                                                                                              |
| ---------------------- | ------------------------- | ------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `max_tokens`           | `int`                     | Maximum tokens per chunk. Defaults to 800. | `DEFAULT_MAX_TOKENS`                                                                                                                                                                                                                                                                                 |
| `subsection_patterns`  | \`list[SubsectionPattern] | None\`                                     | Optional list of SubsectionPattern for detecting implicit headings (e.g., legislative numbering like (a), (1)). When provided, enables enhanced hierarchy extraction.                                                                                                                                |
| `max_subsection_depth` | \`int                     | None\`                                     | Maximum number of subsection levels to recognize. If None (default), uses all patterns in subsection_patterns. Must be between 1 and len(subsection_patterns) if specified.                                                                                                                          |
| `split_on_level`       | \`int                     | None\`                                     | Heading levels \<= this value create new chunks. Levels > this value are accumulated into parent chunks with their markers tracked in subsection_ids. If None (default): - With subsection_patterns: defaults to len(patterns) // 2 - Without patterns: defaults to 6 (split on all markdown levels) |

Raises:

| Type         | Description                                                |
| ------------ | ---------------------------------------------------------- |
| `ValueError` | If max_tokens is not positive.                             |
| `ValueError` | If max_subsection_depth is invalid for the given patterns. |
| `ValueError` | If split_on_level is invalid.                              |

Source code in `src/holodeck/lib/structured_chunker.py`

```
def __init__(
    self,
    max_tokens: int = DEFAULT_MAX_TOKENS,
    subsection_patterns: list[SubsectionPattern] | None = None,
    max_subsection_depth: int | None = None,
    split_on_level: int | None = None,
) -> None:
    """Initialize the structured chunker.

    Args:
        max_tokens: Maximum tokens per chunk. Defaults to 800.
        subsection_patterns: Optional list of SubsectionPattern for detecting
            implicit headings (e.g., legislative numbering like (a), (1)).
            When provided, enables enhanced hierarchy extraction.
        max_subsection_depth: Maximum number of subsection levels to recognize.
            If None (default), uses all patterns in subsection_patterns.
            Must be between 1 and len(subsection_patterns) if specified.
        split_on_level: Heading levels <= this value create new chunks.
            Levels > this value are accumulated into parent chunks with their
            markers tracked in subsection_ids. If None (default):
            - With subsection_patterns: defaults to len(patterns) // 2
            - Without patterns: defaults to 6 (split on all markdown levels)

    Raises:
        ValueError: If max_tokens is not positive.
        ValueError: If max_subsection_depth is invalid for the given patterns.
        ValueError: If split_on_level is invalid.
    """
    if max_tokens <= 0:
        raise ValueError("max_tokens must be positive")

    # Determine effective max depth
    if subsection_patterns is not None:
        max_allowed = len(subsection_patterns)
        if max_subsection_depth is None:
            max_subsection_depth = max_allowed  # Default to all patterns
        elif not 1 <= max_subsection_depth <= max_allowed:
            raise ValueError(
                f"max_subsection_depth must be between 1 and {max_allowed} "
                f"for the given patterns"
            )
    else:
        # No patterns, depth doesn't matter
        max_subsection_depth = max_subsection_depth or 0

    # Determine effective split_on_level
    if split_on_level is not None:
        if split_on_level < 1 or split_on_level > 6:
            raise ValueError("split_on_level must be between 1 and 6")
        effective_split_on_level = split_on_level
    elif subsection_patterns is not None:
        # Default to half of pattern count (rounded down)
        effective_split_on_level = len(subsection_patterns) // 2
    else:
        # No patterns: split on all markdown heading levels
        effective_split_on_level = 6

    self._max_tokens = max_tokens
    self._subsection_patterns = subsection_patterns
    self._max_subsection_depth = max_subsection_depth
    self._split_on_level = effective_split_on_level
    self._encoder = self._initialize_tokenizer()
```

### `max_tokens`

Get the maximum tokens per chunk.

### `is_header_only(chunk)`

Check if chunk contains only a heading with no body content.

Parameters:

| Name    | Type            | Description             | Default    |
| ------- | --------------- | ----------------------- | ---------- |
| `chunk` | `DocumentChunk` | DocumentChunk to check. | *required* |

Returns:

| Type   | Description                                            |
| ------ | ------------------------------------------------------ |
| `bool` | True if chunk is header-only (no substantive content). |

Source code in `src/holodeck/lib/structured_chunker.py`

```
def is_header_only(self, chunk: DocumentChunk) -> bool:
    """Check if chunk contains only a heading with no body content.

    Args:
        chunk: DocumentChunk to check.

    Returns:
        True if chunk is header-only (no substantive content).
    """
    return chunk.chunk_type == ChunkType.HEADER
```

### `parse(markdown, source_path='', mtime=0.0)`

Parse markdown into structure-aware chunks.

Main entry point for document processing. Extracts headings, builds parent chains, and splits content into token-bounded chunks while preserving hierarchical context.

Parameters:

| Name          | Type    | Description                                       | Default    |
| ------------- | ------- | ------------------------------------------------- | ---------- |
| `markdown`    | `str`   | The markdown content to parse.                    | *required* |
| `source_path` | `str`   | Optional path to source file (for metadata).      | `''`       |
| `mtime`       | `float` | Optional file modification time (Unix timestamp). | `0.0`      |

Returns:

| Type                  | Description                                             |
| --------------------- | ------------------------------------------------------- |
| `list[DocumentChunk]` | List of DocumentChunk objects with hierarchy preserved. |

Raises:

| Type         | Description                              |
| ------------ | ---------------------------------------- |
| `ValueError` | If markdown is empty or whitespace-only. |

Example

> > > chunker = StructuredChunker() chunks = chunker.parse("# Title\\n\\nContent", "doc.md") print(chunks[0].section_id) 'sec_title'

Source code in `src/holodeck/lib/structured_chunker.py`

```
def parse(
    self, markdown: str, source_path: str = "", mtime: float = 0.0
) -> list[DocumentChunk]:
    """Parse markdown into structure-aware chunks.

    Main entry point for document processing. Extracts headings,
    builds parent chains, and splits content into token-bounded
    chunks while preserving hierarchical context.

    Args:
        markdown: The markdown content to parse.
        source_path: Optional path to source file (for metadata).
        mtime: Optional file modification time (Unix timestamp).

    Returns:
        List of DocumentChunk objects with hierarchy preserved.

    Raises:
        ValueError: If markdown is empty or whitespace-only.

    Example:
        >>> chunker = StructuredChunker()
        >>> chunks = chunker.parse("# Title\\n\\nContent", "doc.md")
        >>> print(chunks[0].section_id)
        'sec_title'
    """
    if not markdown or not markdown.strip():
        raise ValueError("Markdown content cannot be empty")

    # Extract headings
    headings = self._extract_headings(markdown)

    # If no headings, fall back to flat text chunking
    if not headings:
        return self._parse_flat_text(markdown, source_path, mtime)

    # Split by headings into sections
    sections = self._split_by_headings(markdown, headings)

    # Process each section (split if needed) and collect chunks
    chunks: list[DocumentChunk] = []
    chunk_index = 0

    for section in sections:
        section_chunks = self._split_section(section, source_path, chunk_index)
        # Update mtime on all chunks
        for chunk in section_chunks:
            chunk.mtime = mtime
        chunks.extend(section_chunks)
        chunk_index += len(section_chunks)

    return chunks
```

### Structured Data Loader

Loads and iterates over structured data from CSV, JSON, and JSONL files with field mapping, schema validation, and batch processing for vector store ingestion.

## `StructuredDataLoader(source_path, id_field, vector_fields, metadata_fields=None, field_separator='\n', delimiter=None, batch_size=10000)`

Load and iterate over structured data from CSV, JSON, or JSONL files.

This class provides a unified interface for loading structured data from various file formats and iterating over records with field mapping.

Attributes:

| Name              | Type | Description                                         |
| ----------------- | ---- | --------------------------------------------------- |
| `source_path`     |      | Path to the source data file.                       |
| `id_field`        |      | Field name to use as unique record identifier.      |
| `vector_fields`   |      | List of field names whose values will be embedded.  |
| `metadata_fields` |      | List of field names to include as metadata.         |
| `field_separator` |      | Separator for concatenating multiple vector fields. |
| `delimiter`       |      | CSV delimiter (auto-detected if None).              |
| `batch_size`      |      | Number of records per batch.                        |

Example

> > > loader = StructuredDataLoader( ... source_path="products.csv", ... id_field="id", ... vector_fields=["description"], ... metadata_fields=["title", "category"], ... ) for record in loader.iter_records(): ... print(record["id"], record["content"])

Initialize the StructuredDataLoader.

Parameters:

| Name              | Type        | Description                                         | Default                                                                                                     |
| ----------------- | ----------- | --------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| `source_path`     | `str`       | Path to the source data file.                       | *required*                                                                                                  |
| `id_field`        | `str`       | Field name to use as unique record identifier.      | *required*                                                                                                  |
| `vector_fields`   | `list[str]` | List of field names whose values will be embedded.  | *required*                                                                                                  |
| `metadata_fields` | \`list[str] | None\`                                              | List of field names to include as metadata. If None, includes all fields except id_field and vector_fields. |
| `field_separator` | `str`       | Separator for concatenating multiple vector fields. | `'\n'`                                                                                                      |
| `delimiter`       | \`str       | None\`                                              | CSV delimiter. If None, auto-detected from content.                                                         |
| `batch_size`      | `int`       | Number of records per batch for iter_batches().     | `10000`                                                                                                     |

Source code in `src/holodeck/lib/structured_loader.py`

```
def __init__(
    self,
    source_path: str,
    id_field: str,
    vector_fields: list[str],
    metadata_fields: list[str] | None = None,
    field_separator: str = "\n",
    delimiter: str | None = None,
    batch_size: int = 10_000,
) -> None:
    """Initialize the StructuredDataLoader.

    Args:
        source_path: Path to the source data file.
        id_field: Field name to use as unique record identifier.
        vector_fields: List of field names whose values will be embedded.
        metadata_fields: List of field names to include as metadata.
            If None, includes all fields except id_field and vector_fields.
        field_separator: Separator for concatenating multiple vector fields.
        delimiter: CSV delimiter. If None, auto-detected from content.
        batch_size: Number of records per batch for iter_batches().
    """
    self.source_path = source_path
    self.id_field = id_field
    self.vector_fields = vector_fields
    self.metadata_fields = metadata_fields
    self.field_separator = field_separator
    self.delimiter = delimiter
    self.batch_size = batch_size

    # Cached file type (lazy detection)
    self._file_type: FileType | None = None
    self._detected_delimiter: str | None = None
```

### `file_type`

Get the detected file type (cached).

### `iter_batches()`

Iterate over batches of records.

Groups records into batches of batch_size for efficient processing.

Yields:

| Type                   | Description                                               |
| ---------------------- | --------------------------------------------------------- |
| `list[dict[str, Any]]` | List of record dictionaries (up to batch_size per batch). |

Example

> > > for batch in loader.iter_batches(): ... print(f"Processing {len(batch)} records") ... for record in batch: ... process(record)

Source code in `src/holodeck/lib/structured_loader.py`

```
def iter_batches(self) -> Iterator[list[dict[str, Any]]]:
    """Iterate over batches of records.

    Groups records into batches of batch_size for efficient processing.

    Yields:
        List of record dictionaries (up to batch_size per batch).

    Example:
        >>> for batch in loader.iter_batches():
        ...     print(f"Processing {len(batch)} records")
        ...     for record in batch:
        ...         process(record)
    """
    batch: list[dict[str, Any]] = []

    for record in self.iter_records():
        batch.append(record)

        if len(batch) >= self.batch_size:
            yield batch
            batch = []

    # Yield remaining records
    if batch:
        yield batch
```

### `iter_records()`

Iterate over records with field mapping applied.

Yields dictionaries with keys:

- id: The record identifier (from id_field)
- content: Concatenated vector field values
- metadata: Dictionary of metadata field values

Records with empty content (all vector fields empty) are skipped.

Yields:

| Type             | Description                                     |
| ---------------- | ----------------------------------------------- |
| `dict[str, Any]` | Dictionary with id, content, and metadata keys. |

Example

> > > for record in loader.iter_records(): ... print(record["id"]) ... print(record["content"]) ... print(record["metadata"])

Source code in `src/holodeck/lib/structured_loader.py`

```
def iter_records(self) -> Iterator[dict[str, Any]]:
    """Iterate over records with field mapping applied.

    Yields dictionaries with keys:
    - id: The record identifier (from id_field)
    - content: Concatenated vector field values
    - metadata: Dictionary of metadata field values

    Records with empty content (all vector fields empty) are skipped.

    Yields:
        Dictionary with id, content, and metadata keys.

    Example:
        >>> for record in loader.iter_records():
        ...     print(record["id"])
        ...     print(record["content"])
        ...     print(record["metadata"])
    """
    # Get first record to determine metadata fields
    sample = self._get_sample_record()
    available_fields = list(sample.keys())
    metadata_fields = self._determine_metadata_fields(available_fields)

    skipped_count = 0
    for record in self._get_raw_records():
        # Build content from vector fields
        content = self._build_content(record)

        # Skip records with empty content
        if not content.strip():
            skipped_count += 1
            continue

        # Get record ID (convert to string)
        record_id = str(record.get(self.id_field, ""))

        # Build metadata
        metadata = self._build_metadata(record, metadata_fields)

        yield {
            "id": record_id,
            "content": content,
            "metadata": metadata,
        }

    if skipped_count > 0:
        logger.warning(
            f"Skipped {skipped_count} records with empty vector field content"
        )
```

### `validate_schema()`

Validate that configured fields exist in the source data.

Checks that id_field, vector_fields, and metadata_fields all exist in the source data schema.

Returns:

| Type        | Description                                  |
| ----------- | -------------------------------------------- |
| `list[str]` | List of available field names in the source. |

Raises:

| Type          | Description                                          |
| ------------- | ---------------------------------------------------- |
| `ConfigError` | If any configured field doesn't exist in the source. |

Example

> > > loader = StructuredDataLoader(...) available_fields = loader.validate_schema() print(available_fields) ['id', 'title', 'description', 'category', 'price']

Source code in `src/holodeck/lib/structured_loader.py`

```
def validate_schema(self) -> list[str]:
    """Validate that configured fields exist in the source data.

    Checks that id_field, vector_fields, and metadata_fields all exist
    in the source data schema.

    Returns:
        List of available field names in the source.

    Raises:
        ConfigError: If any configured field doesn't exist in the source.

    Example:
        >>> loader = StructuredDataLoader(...)
        >>> available_fields = loader.validate_schema()
        >>> print(available_fields)
        ['id', 'title', 'description', 'category', 'price']
    """
    sample = self._get_sample_record()
    available_fields = list(sample.keys())

    # Validate id_field
    if self.id_field not in available_fields:
        raise ConfigError(
            field="id_field",
            message=(
                f"Field '{self.id_field}' not found in source. "
                f"Available fields: {available_fields}"
            ),
        )

    # Validate vector_fields
    for field in self.vector_fields:
        if field not in available_fields:
            raise ConfigError(
                field="vector_field",
                message=(
                    f"Field '{field}' not found in source. "
                    f"Available fields: {available_fields}"
                ),
            )

    # Validate metadata_fields (if specified)
    if self.metadata_fields:
        for field in self.metadata_fields:
            if field not in available_fields:
                raise ConfigError(
                    field="metadata_field",
                    message=(
                        f"Field '{field}' not found in source. "
                        f"Available fields: {available_fields}"
                    ),
                )

    return available_fields
```

______________________________________________________________________

## Context Generation

Implements the Anthropic contextual retrieval approach -- generating short context snippets for document chunks to improve semantic search retrieval by 35-49%.

### Claude SDK Context Generator

Uses the Claude Agent SDK `query()` function for contextual embeddings. Supports batched prompts with JSON array parsing and automatic single-chunk fallback.

## `ClaudeContextConfig(model='claude-haiku-4-5', batch_size=10, concurrency=5, max_retries=3, base_delay=1.0, max_document_tokens=8000)`

Configuration for ClaudeSDKContextGenerator.

Attributes:

| Name                  | Type    | Description                                                       |
| --------------------- | ------- | ----------------------------------------------------------------- |
| `model`               | `str`   | Claude model ID for context generation (default: Haiku for cost). |
| `batch_size`          | `int`   | Number of chunks per batch prompt.                                |
| `concurrency`         | `int`   | Maximum concurrent query() calls.                                 |
| `max_retries`         | `int`   | Retry attempts per query.                                         |
| `base_delay`          | `float` | Initial retry delay in seconds.                                   |
| `max_document_tokens` | `int`   | Document truncation limit in tokens.                              |

## `ClaudeSDKContextGenerator(config=None, max_context_tokens=100)`

Generate contextual embeddings using the Claude Agent SDK.

Conforms to the `ContextGenerator` protocol defined in `holodeck.lib.backends.base`.

Uses the Claude Agent SDK `query()` function to call a cheap/fast model (Haiku by default) for generating situating context for document chunks. Supports batched prompts (multiple chunks per call) with JSON parsing and automatic fallback to single-chunk prompts on failure.

Attributes:

| Name                  | Type | Description                           |
| --------------------- | ---- | ------------------------------------- |
| `_config`             |      | Configuration for the generator.      |
| `_max_context_tokens` |      | Maximum tokens for generated context. |
| `_encoder`            |      | Tiktoken encoder for token counting.  |

Initialize the Claude SDK Context Generator.

Parameters:

| Name                 | Type                  | Description                                          | Default                                                         |
| -------------------- | --------------------- | ---------------------------------------------------- | --------------------------------------------------------------- |
| `config`             | \`ClaudeContextConfig | None\`                                               | Configuration for the generator. Uses defaults if not provided. |
| `max_context_tokens` | `int`                 | Maximum tokens for generated context (default: 100). | `100`                                                           |

Source code in `src/holodeck/lib/claude_context_generator.py`

```
def __init__(
    self,
    config: ClaudeContextConfig | None = None,
    max_context_tokens: int = 100,
) -> None:
    """Initialize the Claude SDK Context Generator.

    Args:
        config: Configuration for the generator. Uses defaults if not provided.
        max_context_tokens: Maximum tokens for generated context (default: 100).
    """
    self._config = config or ClaudeContextConfig()
    self._max_context_tokens = max_context_tokens
    self._encoder = tiktoken.get_encoding("cl100k_base")
```

### `contextualize_batch(chunks, document_text, concurrency=None)`

Generate contextual descriptions for a batch of chunks.

Splits chunks into sub-batches of `config.batch_size`, processes them concurrently (bounded by semaphore), and returns results in order.

Parameters:

| Name            | Type                  | Description                       | Default                                                    |
| --------------- | --------------------- | --------------------------------- | ---------------------------------------------------------- |
| `chunks`        | `list[DocumentChunk]` | Document chunks to contextualize. | *required*                                                 |
| `document_text` | `str`                 | Full text of the source document. | *required*                                                 |
| `concurrency`   | \`int                 | None\`                            | Maximum concurrent LLM calls. Uses config default if None. |

Returns:

| Type        | Description                                              |
| ----------- | -------------------------------------------------------- |
| `list[str]` | A list of contextualized content strings, one per chunk. |

Source code in `src/holodeck/lib/claude_context_generator.py`

```
async def contextualize_batch(
    self,
    chunks: list[DocumentChunk],
    document_text: str,
    concurrency: int | None = None,
) -> list[str]:
    """Generate contextual descriptions for a batch of chunks.

    Splits chunks into sub-batches of ``config.batch_size``, processes them
    concurrently (bounded by semaphore), and returns results in order.

    Args:
        chunks: Document chunks to contextualize.
        document_text: Full text of the source document.
        concurrency: Maximum concurrent LLM calls. Uses config default if None.

    Returns:
        A list of contextualized content strings, one per chunk.
    """
    if not chunks:
        return []

    effective_concurrency = concurrency or self._config.concurrency
    semaphore = asyncio.Semaphore(effective_concurrency)
    batch_size = self._config.batch_size

    # Split into sub-batches
    sub_batches = [
        chunks[i : i + batch_size] for i in range(0, len(chunks), batch_size)
    ]

    async def process_with_semaphore(
        batch: list[DocumentChunk],
    ) -> list[str]:
        async with semaphore:
            return await self._process_batch(batch, document_text)

    # Process all sub-batches concurrently
    batch_results = await asyncio.gather(
        *(process_with_semaphore(batch) for batch in sub_batches)
    )

    # Flatten results in order
    return [item for sublist in batch_results for item in sublist]
```

### LLM Context Generator

Uses Semantic Kernel chat completion services for contextual embeddings. Supports adaptive concurrency on rate limiting and exponential backoff retry logic.

## `RetryConfig(max_retries=3, base_delay=1.0, exponential_base=2.0, max_delay=10.0)`

Configuration for exponential backoff retry logic.

Attributes:

| Name               | Type    | Description                                                 |
| ------------------ | ------- | ----------------------------------------------------------- |
| `max_retries`      | `int`   | Maximum number of retry attempts (default: 3).              |
| `base_delay`       | `float` | Initial delay in seconds before first retry (default: 1.0). |
| `exponential_base` | `float` | Multiplier for exponential backoff (default: 2.0).          |
| `max_delay`        | `float` | Maximum delay cap in seconds (default: 10.0).               |

Example

With defaults, delays are: 1s, 2s, 4s (capped at max_delay if exceeded).

## `LLMContextGenerator(chat_service, execution_settings=None, max_context_tokens=DEFAULT_MAX_CONTEXT_TOKENS, max_document_tokens=DEFAULT_MAX_DOCUMENT_TOKENS, concurrency=None, retry_config=None)`

Generate contextual embeddings using LLM (Anthropic approach).

Conforms to the `ContextGenerator` protocol defined in `holodeck.lib.backends.base`.

This class generates short context snippets for document chunks to improve semantic search retrieval. It follows Anthropic's contextual retrieval approach which prepends situational context to each chunk before embedding.

Attributes:

| Name                   | Type | Description                                            |
| ---------------------- | ---- | ------------------------------------------------------ |
| `_chat_service`        |      | Semantic Kernel chat completion service.               |
| `_max_context_tokens`  |      | Maximum tokens for generated context (default: 100).   |
| `_max_document_tokens` |      | Maximum tokens for document in prompt (default: 8000). |
| `_concurrency`         |      | Current concurrency limit for batch processing.        |
| `_retry_config`        |      | Configuration for exponential backoff retries.         |

Example

> > > from semantic_kernel.connectors.ai.open_ai import OpenAIChatCompletion chat_service = OpenAIChatCompletion(ai_model_id="gpt-4o-mini") generator = LLMContextGenerator(chat_service=chat_service) context = await generator.generate_context(chunk_text, document_text)

Initialize the LLM Context Generator.

Parameters:

| Name                  | Type                       | Description                                             | Default                                                                                                               |
| --------------------- | -------------------------- | ------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| `chat_service`        | `ChatCompletionClientBase` | Semantic Kernel chat completion service instance.       | *required*                                                                                                            |
| `execution_settings`  | \`PromptExecutionSettings  | None\`                                                  | Optional prompt execution settings. If not provided, defaults will be used with max_tokens set to max_context_tokens. |
| `max_context_tokens`  | `int`                      | Maximum tokens for generated context (default: 100).    | `DEFAULT_MAX_CONTEXT_TOKENS`                                                                                          |
| `max_document_tokens` | `int`                      | Maximum tokens for document truncation (default: 8000). | `DEFAULT_MAX_DOCUMENT_TOKENS`                                                                                         |
| `concurrency`         | \`int                      | None\`                                                  | Maximum concurrent LLM requests (default: 10).                                                                        |
| `retry_config`        | \`RetryConfig              | None\`                                                  | Configuration for retry logic. Uses defaults if not provided.                                                         |

Source code in `src/holodeck/lib/llm_context_generator.py`

```
def __init__(
    self,
    chat_service: "ChatCompletionClientBase",
    execution_settings: "PromptExecutionSettings | None" = None,
    max_context_tokens: int = DEFAULT_MAX_CONTEXT_TOKENS,
    max_document_tokens: int = DEFAULT_MAX_DOCUMENT_TOKENS,
    concurrency: int | None = None,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize the LLM Context Generator.

    Args:
        chat_service: Semantic Kernel chat completion service instance.
        execution_settings: Optional prompt execution settings. If not provided,
            defaults will be used with max_tokens set to max_context_tokens.
        max_context_tokens: Maximum tokens for generated context (default: 100).
        max_document_tokens: Maximum tokens for document truncation (default: 8000).
        concurrency: Maximum concurrent LLM requests (default: 10).
        retry_config: Configuration for retry logic. Uses defaults if not provided.
    """
    self._chat_service = chat_service
    self._execution_settings = execution_settings
    self._max_context_tokens = max_context_tokens
    self._max_document_tokens = max_document_tokens
    self._concurrency = (
        concurrency if concurrency is not None else self.DEFAULT_CONCURRENCY
    )
    self._retry_config = retry_config or RetryConfig()
    self._encoder = tiktoken.get_encoding("cl100k_base")
```

### `contextualize_batch(chunks, document_text, concurrency=None)`

Batch process multiple chunks with concurrency control.

Processes all chunks concurrently using a semaphore to limit the number of simultaneous LLM calls. Results maintain the same order as input chunks.

Parameters:

| Name            | Type                  | Description                              | Default                                                       |
| --------------- | --------------------- | ---------------------------------------- | ------------------------------------------------------------- |
| `chunks`        | `list[DocumentChunk]` | List of DocumentChunks to contextualize. | *required*                                                    |
| `document_text` | `str`                 | The full document text for context.      | *required*                                                    |
| `concurrency`   | \`int                 | None\`                                   | Optional concurrency override. Uses instance default if None. |

Returns:

| Type        | Description                                                           |
| ----------- | --------------------------------------------------------------------- |
| `list[str]` | List of contextualized content strings in same order as input chunks. |

Example

> > > results = await generator.contextualize_batch(chunks, document) assert len(results) == len(chunks)

Source code in `src/holodeck/lib/llm_context_generator.py`

```
async def contextualize_batch(
    self,
    chunks: list[DocumentChunk],
    document_text: str,
    concurrency: int | None = None,
) -> list[str]:
    """Batch process multiple chunks with concurrency control.

    Processes all chunks concurrently using a semaphore to limit the number
    of simultaneous LLM calls. Results maintain the same order as input chunks.

    Args:
        chunks: List of DocumentChunks to contextualize.
        document_text: The full document text for context.
        concurrency: Optional concurrency override. Uses instance default if None.

    Returns:
        List of contextualized content strings in same order as input chunks.

    Example:
        >>> results = await generator.contextualize_batch(chunks, document)
        >>> assert len(results) == len(chunks)
    """
    if not chunks:
        return []

    effective_concurrency = concurrency or self._concurrency
    semaphore = asyncio.Semaphore(effective_concurrency)

    async def process_chunk(chunk: DocumentChunk) -> str:
        async with semaphore:
            return await self.contextualize_chunk(chunk, document_text)

    # Use gather to maintain order
    tasks = [process_chunk(chunk) for chunk in chunks]
    results = await asyncio.gather(*tasks)

    return list(results)
```

### `contextualize_chunk(chunk, document_text)`

Contextualize a single document chunk.

Generates context for the chunk and prepends it to the chunk content in the format: "{context}\\n\\n{chunk.content}".

Parameters:

| Name            | Type            | Description                         | Default    |
| --------------- | --------------- | ----------------------------------- | ---------- |
| `chunk`         | `DocumentChunk` | The DocumentChunk to contextualize. | *required* |
| `document_text` | `str`           | The full document text for context. | *required* |

Returns:

| Type  | Description                                                          |
| ----- | -------------------------------------------------------------------- |
| `str` | Contextualized content string. On failure, returns original content. |

Example

> > > result = await generator.contextualize_chunk(chunk, document) print(result) "This chunk defines force majeure terms.

Force Majeure means any event beyond reasonable control..."

Source code in `src/holodeck/lib/llm_context_generator.py`

```
async def contextualize_chunk(
    self, chunk: DocumentChunk, document_text: str
) -> str:
    """Contextualize a single document chunk.

    Generates context for the chunk and prepends it to the chunk content
    in the format: "{context}\\n\\n{chunk.content}".

    Args:
        chunk: The DocumentChunk to contextualize.
        document_text: The full document text for context.

    Returns:
        Contextualized content string. On failure, returns original content.

    Example:
        >>> result = await generator.contextualize_chunk(chunk, document)
        >>> print(result)
        "This chunk defines force majeure terms.

        Force Majeure means any event beyond reasonable control..."
    """
    context = await self.generate_context(chunk.content, document_text)

    if context:
        return f"{context}\n\n{chunk.content}"
    else:
        # Graceful degradation - return original content
        return chunk.content
```

### `generate_context(chunk_text, document_text)`

Generate context for a single chunk.

Uses the LLM to generate a short (50-100 token) context snippet that situates the chunk within the broader document. Includes retry logic with exponential backoff for resilience.

Parameters:

| Name            | Type  | Description                                | Default    |
| --------------- | ----- | ------------------------------------------ | ---------- |
| `chunk_text`    | `str` | The chunk content to generate context for. | *required* |
| `document_text` | `str` | The full document text for context.        | *required* |

Returns:

| Type  | Description                                           |
| ----- | ----------------------------------------------------- |
| `str` | Generated context string, or empty string on failure. |

Example

> > > context = await generator.generate_context( ... "Force Majeure means any event...", ... full_policy_document ... ) print(context) "This is the definition section of the insurance policy."

Source code in `src/holodeck/lib/llm_context_generator.py`

```
async def generate_context(self, chunk_text: str, document_text: str) -> str:
    """Generate context for a single chunk.

    Uses the LLM to generate a short (50-100 token) context snippet that
    situates the chunk within the broader document. Includes retry logic
    with exponential backoff for resilience.

    Args:
        chunk_text: The chunk content to generate context for.
        document_text: The full document text for context.

    Returns:
        Generated context string, or empty string on failure.

    Example:
        >>> context = await generator.generate_context(
        ...     "Force Majeure means any event...",
        ...     full_policy_document
        ... )
        >>> print(context)
        "This is the definition section of the insurance policy."
    """
    prompt = self._format_prompt(chunk_text, document_text)

    for attempt in range(self._retry_config.max_retries):
        try:
            return await self._call_llm(prompt)
        except Exception as e:
            is_last_attempt = attempt == self._retry_config.max_retries - 1

            if is_last_attempt:
                max_retries = self._retry_config.max_retries
                logger.warning(
                    f"Context generation failed after {max_retries} attempts: {e}"
                )
                return ""  # Graceful degradation

            # Handle rate limiting
            if self._is_rate_limit_error(e):
                self._reduce_concurrency()

            delay = self._get_retry_delay(attempt, e)
            logger.debug(
                f"Retry attempt {attempt + 1}/{self._retry_config.max_retries} "
                f"after {delay:.2f}s delay"
            )
            await asyncio.sleep(delay)

    return ""  # Should not reach here, but for safety
```

______________________________________________________________________

## Tool Initialization

Shared tool initialization for VectorStoreTool and HierarchicalDocumentTool. Provider-agnostic: works for both SK and Claude backend paths.

## `resolve_embedding_model(agent)`

Resolve embedding model name from agent config.

Checks vectorstore and hierarchical-doc tool configs for explicit `embedding_model` values first. If explicit values conflict across tools, raises an error because embedding services are shared per agent. Falls back to provider defaults when no explicit value is configured.

Parameters:

| Name    | Type    | Description          | Default    |
| ------- | ------- | -------------------- | ---------- |
| `agent` | `Agent` | Agent configuration. | *required* |

Returns:

| Type  | Description                  |
| ----- | ---------------------------- |
| `str` | Embedding model name string. |

Raises:

| Type                   | Description                                  |
| ---------------------- | -------------------------------------------- |
| `ToolInitializerError` | If explicit embedding_model values conflict. |

Source code in `src/holodeck/lib/tool_initializer.py`

```
def resolve_embedding_model(agent: Agent) -> str:
    """Resolve embedding model name from agent config.

    Checks vectorstore and hierarchical-doc tool configs for explicit
    ``embedding_model`` values first. If explicit values conflict across tools,
    raises an error because embedding services are shared per agent. Falls back
    to provider defaults when no explicit value is configured.

    Args:
        agent: Agent configuration.

    Returns:
        Embedding model name string.

    Raises:
        ToolInitializerError: If explicit embedding_model values conflict.
    """
    from holodeck.models.tool import (
        HierarchicalDocumentToolConfig,
        VectorstoreTool,
    )

    explicit_models: dict[str, str] = {}
    if agent.tools:
        for tool in agent.tools:
            model_name: str | None = None
            if isinstance(tool, VectorstoreTool | HierarchicalDocumentToolConfig):
                model_name = tool.embedding_model

            if model_name:
                explicit_models[tool.name] = model_name

    if explicit_models:
        unique_models = sorted(set(explicit_models.values()))
        if len(unique_models) > 1:
            configured_models = ", ".join(
                f"{tool_name}={model_name}"
                for tool_name, model_name in sorted(explicit_models.items())
            )
            raise ToolInitializerError(
                "Conflicting embedding_model values detected across vectorstore/"
                "hierarchical_document tools. A single shared embedding service "
                "is used per agent, so explicit embedding_model values must match. "
                f"Found: {configured_models}"
            )
        return unique_models[0]

    # Determine which provider to use for defaults
    provider = _resolve_embedding_provider(agent)

    if provider == ProviderEnum.OLLAMA:
        return "nomic-embed-text:latest"
    # OpenAI / Azure OpenAI default
    return "text-embedding-3-small"
```

## `create_embedding_service(agent)`

Create an SK TextEmbedding service from agent config.

For Anthropic provider: uses `agent.embedding_provider` config. For OpenAI/Azure/Ollama: uses `agent.model` config directly.

Parameters:

| Name    | Type    | Description          | Default    |
| ------- | ------- | -------------------- | ---------- |
| `agent` | `Agent` | Agent configuration. | *required* |

Returns:

| Type  | Description                                    |
| ----- | ---------------------------------------------- |
| `Any` | An initialized TextEmbedding service instance. |

Raises:

| Type                   | Description                             |
| ---------------------- | --------------------------------------- |
| `ToolInitializerError` | If provider doesn't support embeddings. |

Source code in `src/holodeck/lib/tool_initializer.py`

```
def create_embedding_service(agent: Agent) -> Any:
    """Create an SK TextEmbedding service from agent config.

    For Anthropic provider: uses ``agent.embedding_provider`` config.
    For OpenAI/Azure/Ollama: uses ``agent.model`` config directly.

    Args:
        agent: Agent configuration.

    Returns:
        An initialized TextEmbedding service instance.

    Raises:
        ToolInitializerError: If provider doesn't support embeddings.
    """
    from semantic_kernel.connectors.ai.open_ai import (
        AzureTextEmbedding,
        OpenAITextEmbedding,
    )

    provider = _resolve_embedding_provider(agent)
    model_config = _resolve_embedding_model_config(agent)
    embedding_model = resolve_embedding_model(agent)

    logger.debug(
        "Creating embedding service: model=%s, provider=%s",
        embedding_model,
        provider,
    )

    api_key_raw = (
        model_config.api_key.get_secret_value()
        if model_config.api_key is not None
        else None
    )

    if provider == ProviderEnum.OPENAI:
        return OpenAITextEmbedding(
            ai_model_id=embedding_model,
            api_key=api_key_raw,
        )

    if provider == ProviderEnum.AZURE_OPENAI:
        azure_embed_kwargs: dict[str, Any] = {
            "deployment_name": embedding_model,
            "endpoint": model_config.endpoint,
            "api_key": api_key_raw,
            "api_version": model_config.api_version or AZURE_OPENAI_DEFAULT_API_VERSION,
        }
        return AzureTextEmbedding(**azure_embed_kwargs)

    if provider == ProviderEnum.OLLAMA:
        try:
            from semantic_kernel.connectors.ai.ollama import OllamaTextEmbedding
        except ImportError as exc:
            raise ToolInitializerError(
                "Ollama provider requires 'ollama' package. "
                "Install with: pip install ollama"
            ) from exc

        return OllamaTextEmbedding(
            ai_model_id=embedding_model,
            host=model_config.endpoint if model_config.endpoint else None,
        )

    raise ToolInitializerError(
        f"Embedding service not supported for provider: {provider}. "
        "Vectorstore tools require OpenAI, Azure OpenAI, or Ollama provider."
    )
```

## `initialize_tools(agent, force_ingest=False, execution_config=None, chat_service=None, base_dir=None, context_generator=None)`

Initialize all vectorstore and hierarchical-doc tools for an agent.

Creates embedding service, initializes each tool, returns dict keyed by tool config name. This is the main entry point for both backends.

Parameters:

| Name                | Type              | Description                                      | Default                                                                                                        |
| ------------------- | ----------------- | ------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| `agent`             | `Agent`           | Agent configuration.                             | *required*                                                                                                     |
| `force_ingest`      | `bool`            | Force re-ingestion of vector store source files. | `False`                                                                                                        |
| `execution_config`  | \`ExecutionConfig | None\`                                           | Execution configuration for file processing.                                                                   |
| `chat_service`      | \`Any             | None\`                                           | Optional chat service for hierarchical doc tools.                                                              |
| `base_dir`          | \`str             | None\`                                           | Base directory for resolving relative source paths. If None, falls back to agent_base_dir context variable.    |
| `context_generator` | \`Any             | None\`                                           | Optional pre-built ContextGenerator instance. When provided, takes highest priority for contextual embeddings. |

Returns:

| Type             | Description                                          |
| ---------------- | ---------------------------------------------------- |
| `dict[str, Any]` | Dict mapping tool name to initialized tool instance. |

Raises:

| Type                   | Description |
| ---------------------- | ----------- |
| `ToolInitializerError` | On failure. |

Source code in `src/holodeck/lib/tool_initializer.py`

```
async def initialize_tools(
    agent: Agent,
    force_ingest: bool = False,
    execution_config: ExecutionConfig | None = None,
    chat_service: Any | None = None,
    base_dir: str | None = None,
    context_generator: Any | None = None,
) -> dict[str, Any]:
    """Initialize all vectorstore and hierarchical-doc tools for an agent.

    Creates embedding service, initializes each tool, returns dict keyed by
    tool config name. This is the main entry point for both backends.

    Args:
        agent: Agent configuration.
        force_ingest: Force re-ingestion of vector store source files.
        execution_config: Execution configuration for file processing.
        chat_service: Optional chat service for hierarchical doc tools.
        base_dir: Base directory for resolving relative source paths.
            If None, falls back to agent_base_dir context variable.
        context_generator: Optional pre-built ContextGenerator instance.
            When provided, takes highest priority for contextual embeddings.

    Returns:
        Dict mapping tool name to initialized tool instance.

    Raises:
        ToolInitializerError: On failure.
    """
    from holodeck.models.tool import (
        HierarchicalDocumentToolConfig,
    )
    from holodeck.models.tool import (
        VectorstoreTool as VectorstoreToolConfig,
    )

    if not agent.tools:
        return {}

    has_vs = any(isinstance(t, VectorstoreToolConfig) for t in agent.tools)
    has_hd = any(isinstance(t, HierarchicalDocumentToolConfig) for t in agent.tools)

    if not has_vs and not has_hd:
        return {}

    try:
        embedding_service = create_embedding_service(agent)
    except Exception as exc:
        raise ToolInitializerError(
            f"Failed to create embedding service: {exc}"
        ) from exc

    instances: dict[str, Any] = {}

    # Get provider type for dimension resolution
    provider_type = _resolve_embedding_provider(agent).value

    # Resolve base_dir: explicit > context var
    effective_base_dir = base_dir
    if effective_base_dir is None:
        from holodeck.config.context import agent_base_dir

        effective_base_dir = agent_base_dir.get()

    # Initialize vectorstore tools
    if has_vs:
        vs_instances = await _initialize_vectorstore_tools(
            agent=agent,
            embedding_service=embedding_service,
            force_ingest=force_ingest,
            execution_config=execution_config,
            provider_type=provider_type,
            base_dir=effective_base_dir,
        )
        instances.update(vs_instances)

    # Initialize hierarchical document tools
    if has_hd:
        hd_instances = await initialize_hierarchical_doc_tools(
            agent=agent,
            embedding_service=embedding_service,
            chat_service=chat_service,
            force_ingest=force_ingest,
            provider_type=provider_type,
            base_dir=effective_base_dir,
            context_generator=context_generator,
            execution_config=execution_config,
        )
        instances.update(hd_instances)

    return instances
```

## `initialize_hierarchical_doc_tools(agent, embedding_service, chat_service, force_ingest, provider_type, base_dir=None, context_generator=None, execution_config=None)`

Initialize all hierarchical document tools from agent config.

Parameters:

| Name                | Type              | Description                                    | Default                                                                                                                                                                   |
| ------------------- | ----------------- | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent`             | `Agent`           | Agent configuration.                           | *required*                                                                                                                                                                |
| `embedding_service` | `Any`             | SK TextEmbedding service.                      | *required*                                                                                                                                                                |
| `chat_service`      | \`Any             | None\`                                         | Optional chat service for context generation.                                                                                                                             |
| `force_ingest`      | `bool`            | Force re-ingestion of source files.            | *required*                                                                                                                                                                |
| `provider_type`     | `str`             | Provider type string for dimension resolution. | *required*                                                                                                                                                                |
| `base_dir`          | \`str             | None\`                                         | Base directory for resolving relative source paths.                                                                                                                       |
| `context_generator` | \`Any             | None\`                                         | Optional pre-built ContextGenerator instance.                                                                                                                             |
| `execution_config`  | \`ExecutionConfig | None\`                                         | Execution configuration forwarded to each tool so execution.file_timeout (and download_timeout / cache_dir) from agent.yaml reaches the FileProcessor used during ingest. |

Returns:

| Type             | Description                                                              |
| ---------------- | ------------------------------------------------------------------------ |
| `dict[str, Any]` | Dict mapping tool name to initialized HierarchicalDocumentTool instance. |

Raises:

| Type                   | Description                      |
| ---------------------- | -------------------------------- |
| `ToolInitializerError` | If any tool fails to initialize. |

Source code in `src/holodeck/lib/tool_initializer.py`

```
async def initialize_hierarchical_doc_tools(
    agent: Agent,
    embedding_service: Any,
    chat_service: Any | None,
    force_ingest: bool,
    provider_type: str,
    base_dir: str | None = None,
    context_generator: Any | None = None,
    execution_config: ExecutionConfig | None = None,
) -> dict[str, Any]:
    """Initialize all hierarchical document tools from agent config.

    Args:
        agent: Agent configuration.
        embedding_service: SK TextEmbedding service.
        chat_service: Optional chat service for context generation.
        force_ingest: Force re-ingestion of source files.
        provider_type: Provider type string for dimension resolution.
        base_dir: Base directory for resolving relative source paths.
        context_generator: Optional pre-built ContextGenerator instance.
        execution_config: Execution configuration forwarded to each tool so
            ``execution.file_timeout`` (and download_timeout / cache_dir)
            from agent.yaml reaches the FileProcessor used during ingest.

    Returns:
        Dict mapping tool name to initialized HierarchicalDocumentTool instance.

    Raises:
        ToolInitializerError: If any tool fails to initialize.
    """
    from holodeck.models.tool import HierarchicalDocumentToolConfig
    from holodeck.tools.hierarchical_document_tool import HierarchicalDocumentTool

    instances: dict[str, Any] = {}
    cleanup_stack: list[Path] = []

    try:
        for tool_config in agent.tools or []:
            if not isinstance(tool_config, HierarchicalDocumentToolConfig):
                continue

            try:
                # Resolve remote source if needed
                effective_config = tool_config
                is_remote = _is_remote_source(tool_config.source)
                resolved_local_path: Path | None = None
                if is_remote:
                    resolved = await SourceResolver.resolve(
                        tool_config.source, base_dir
                    )
                    if resolved.temp_dir:
                        cleanup_stack.append(resolved.temp_dir)
                    resolved_local_path = resolved.local_path
                    effective_config = tool_config.model_copy(
                        update={"source": str(resolved_local_path)}
                    )

                tool = HierarchicalDocumentTool(
                    effective_config,
                    base_dir=base_dir,
                    execution_config=execution_config,
                )
                if is_remote and resolved_local_path is not None:
                    tool.set_source_context(
                        source_root=resolved_local_path,
                        is_remote=True,
                    )
                tool.set_embedding_service(embedding_service)

                # Resolve context generator via 5-tier priority chain
                resolved_generator = _resolve_context_generator(
                    agent=agent,
                    tool_config=tool_config,
                    context_generator=context_generator,
                    chat_service=chat_service,
                )
                if resolved_generator is not None:
                    tool.set_context_generator(resolved_generator)

                await tool.initialize(
                    force_ingest=force_ingest, provider_type=provider_type
                )
                instances[tool_config.name] = tool
                logger.info(
                    "Initialized hierarchical document tool: %s",
                    tool_config.name,
                )
            except Exception as exc:
                raise ToolInitializerError(
                    f"Failed to initialize hierarchical document tool "
                    f"'{tool_config.name}': {exc}"
                ) from exc
    except Exception:
        for temp_dir in cleanup_stack:
            await SourceResolver.cleanup(temp_dir)
        raise

    return instances
```

______________________________________________________________________

## Instruction Resolution

Resolves agent instructions from `Instructions` config objects, supporting both inline text and file-based instructions with base directory resolution.

## `resolve_instructions(instructions, base_dir=None)`

Resolve agent instructions from an `Instructions` config object.

Parameters:

| Name           | Type           | Description                                               | Default                                                                                                                 |
| -------------- | -------------- | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `instructions` | `Instructions` | Instructions config with either inline text or file path. | *required*                                                                                                              |
| `base_dir`     | \`Path         | None\`                                                    | Explicit base directory for resolving relative file paths. Falls back to the agent_base_dir context variable, then CWD. |

Returns:

| Type  | Description                    |
| ----- | ------------------------------ |
| `str` | The resolved instruction text. |

Raises:

| Type          | Description                                            |
| ------------- | ------------------------------------------------------ |
| `ConfigError` | If the instructions file is missing or cannot be read. |

Source code in `src/holodeck/lib/instruction_resolver.py`

```
def resolve_instructions(
    instructions: Instructions, base_dir: Path | None = None
) -> str:
    """Resolve agent instructions from an ``Instructions`` config object.

    Args:
        instructions: Instructions config with either ``inline`` text or ``file`` path.
        base_dir: Explicit base directory for resolving relative file paths.
            Falls back to the ``agent_base_dir`` context variable, then CWD.

    Returns:
        The resolved instruction text.

    Raises:
        ConfigError: If the instructions file is missing or cannot be read.
    """
    if instructions.inline:
        return instructions.inline

    if instructions.file:
        resolved_base = base_dir or _resolve_base_dir()
        file_path = (
            resolved_base / instructions.file
            if resolved_base
            else Path(instructions.file)
        )

        if not file_path.exists():
            raise ConfigError(
                "instructions.file", f"Instructions file not found: {file_path}"
            )

        try:
            return file_path.read_text()
        except OSError as exc:
            raise ConfigError(
                "instructions.file", f"Failed to read instructions file: {exc}"
            ) from exc

    raise ConfigError("instructions", "No instructions provided (file or inline)")
```

______________________________________________________________________

## Vector Store

Unified interface for working with various vector storage backends through Semantic Kernel's VectorStoreCollection abstractions. Supports PostgreSQL (pgvector), Azure AI Search, Qdrant, Weaviate, ChromaDB, FAISS, Pinecone, and more.

## `ChromaConnectionParams`

Bases: `TypedDict`

Parameters for ChromaDB connection.

Attributes:

| Name   | Type   | Description                         |
| ------ | ------ | ----------------------------------- |
| `host` | `str`  | Server hostname (e.g., 'localhost') |
| `port` | `int`  | Server port (e.g., 8000)            |
| `ssl`  | `bool` | Whether to use HTTPS                |

## `QdrantConnectionParams`

Bases: `TypedDict`

Parameters for Qdrant connection.

All fields are optional since Qdrant defaults to in-memory when no params provided.

Attributes:

| Name          | Type   | Description                      |
| ------------- | ------ | -------------------------------- |
| `url`         | \`str  | None\`                           |
| `api_key`     | \`str  | None\`                           |
| `host`        | \`str  | None\`                           |
| `port`        | \`int  | None\`                           |
| `grpc_port`   | \`int  | None\`                           |
| `prefer_grpc` | `bool` | Whether to prefer gRPC over HTTP |
| `location`    | \`str  | None\`                           |
| `path`        | \`str  | None\`                           |

## `PineconeConnectionParams`

Bases: `TypedDict`

Parameters for Pinecone connection.

Attributes:

| Name        | Type   | Description                                 |
| ----------- | ------ | ------------------------------------------- |
| `api_key`   | \`str  | None\`                                      |
| `namespace` | \`str  | None\`                                      |
| `use_grpc`  | `bool` | Whether to use gRPC client (default: False) |

## `PostgresConnectionParams`

Bases: `TypedDict`

Parameters for PostgreSQL connection.

Attributes:

| Name                | Type  | Description |
| ------------------- | ----- | ----------- |
| `connection_string` | \`str | None\`      |
| `host`              | \`str | None\`      |
| `port`              | \`int | None\`      |
| `dbname`            | \`str | None\`      |
| `user`              | \`str | None\`      |
| `password`          | \`str | None\`      |
| `sslmode`           | \`str | None\`      |
| `db_schema`         | \`str | None\`      |

## `QueryResult`

Search result from vector store query.

Represents a single match returned from semantic search operations.

Attributes:

| Name          | Type             | Description                                             |
| ------------- | ---------------- | ------------------------------------------------------- |
| `content`     | `str`            | Matched document chunk content                          |
| `score`       | `float`          | Relevance/similarity score (0.0-1.0, higher is better)  |
| `source_path` | `str`            | Original source file path                               |
| `chunk_index` | `int`            | Chunk index within source file                          |
| `metadata`    | `dict[str, Any]` | Additional metadata (file_type, file_size, mtime, etc.) |

## `StructuredQueryResult`

Search result from structured data vector store query.

Represents a single match returned from semantic search over structured data (CSV, JSON, JSONL files). Unlike QueryResult which uses chunk_index, this uses the original record ID from the source data.

Attributes:

| Name          | Type             | Description                                                 |
| ------------- | ---------------- | ----------------------------------------------------------- |
| `id`          | `str`            | Original record identifier from the id_field in source data |
| `content`     | `str`            | Concatenated vector field content that was embedded         |
| `score`       | `float`          | Relevance/similarity score (0.0-1.0, higher is better)      |
| `source_file` | `str`            | Original source file path (e.g., "products.csv")            |
| `metadata`    | `dict[str, Any]` | Dictionary of metadata field values from the source record  |

## `create_document_record_class(dimensions=1536)`

Create a DocumentRecord class with specified embedding dimensions.

This factory creates a new DocumentRecord dataclass with custom dimensions. Each collection can have its own DocumentRecord type.

Parameters:

| Name         | Type  | Description                 | Default |
| ------------ | ----- | --------------------------- | ------- |
| `dimensions` | `int` | Embedding vector dimensions | `1536`  |

Returns:

| Type        | Description                                                  |
| ----------- | ------------------------------------------------------------ |
| `type[Any]` | DocumentRecord class configured for the specified dimensions |

Raises:

| Type         | Description              |
| ------------ | ------------------------ |
| `ValueError` | If dimensions is invalid |

Source code in `src/holodeck/lib/vector_store.py`

```
def create_document_record_class(dimensions: int = 1536) -> type[Any]:
    """Create a DocumentRecord class with specified embedding dimensions.

    This factory creates a new DocumentRecord dataclass with custom dimensions.
    Each collection can have its own DocumentRecord type.

    Args:
        dimensions: Embedding vector dimensions

    Returns:
        DocumentRecord class configured for the specified dimensions

    Raises:
        ValueError: If dimensions is invalid
    """
    if dimensions <= 0 or dimensions > 10000:
        raise ValueError(f"Invalid dimensions: {dimensions}")

    @vectorstoremodel(collection_name=f"documents_dim{dimensions}")
    @dataclass
    class DynamicDocumentRecord:  # type: ignore[misc]
        """Vector store record for document chunks with embeddings.

        Each document file is split into multiple chunks, each with its own embedding.
        This record is compatible with all Semantic Kernel vector store backends.

        The @vectorstoremodel decorator enables automatic schema generation for the
        underlying vector database, supporting all major vector store providers.

        Attributes:
            id: Unique identifier (key field) following format:
                {source_path}_chunk_{chunk_index}
            source_path: Original source file path (indexed for filtering)
            chunk_index: Chunk index within document (0-indexed, indexed)
            content: Chunk content for semantic search (full-text indexed)
            embedding: Vector embedding
            mtime: File modification time (Unix timestamp) for change detection
            file_type: Source file extension (.txt, .md, .pdf, etc.)
            file_size_bytes: Original file size in bytes
        """

        id: Annotated[str, VectorStoreField("key")] = field(
            default_factory=lambda: str(uuid4())
        )
        source_path: Annotated[str, VectorStoreField("data", is_indexed=True)] = field(
            default=""
        )
        chunk_index: Annotated[int, VectorStoreField("data", is_indexed=True)] = field(
            default=0
        )
        content: Annotated[str, VectorStoreField("data", is_full_text_indexed=True)] = (
            field(default="")
        )
        embedding: Annotated[
            list[float] | None,
            VectorStoreField(
                "vector",
                dimensions=dimensions,
                distance_function=DistanceFunction.COSINE_SIMILARITY,
            ),
        ] = field(default=None)
        mtime: Annotated[float, VectorStoreField("data")] = field(default=0.0)
        file_type: Annotated[str, VectorStoreField("data")] = field(default="")
        file_size_bytes: Annotated[int, VectorStoreField("data")] = field(default=0)
        content_hash: Annotated[str, VectorStoreField("data")] = field(default="")

    return cast(type[Any], DynamicDocumentRecord)
```

## `create_structured_record_class(dimensions=1536, metadata_field_names=None, collection_name='structured_records')`

Create a StructuredRecord class and definition for structured data.

This factory creates a new StructuredRecord dataclass AND a matching VectorStoreCollectionDefinition. Both are needed for proper persistence of dynamic metadata fields to Semantic Kernel vector stores.

Unlike DocumentRecord which is for unstructured documents, this is designed for structured data (CSV, JSON, JSONL) with user-defined metadata fields.

Parameters:

| Name                   | Type        | Description                                 | Default                                 |
| ---------------------- | ----------- | ------------------------------------------- | --------------------------------------- |
| `dimensions`           | `int`       | Embedding vector dimensions (default: 1536) | `1536`                                  |
| `metadata_field_names` | \`list[str] | None\`                                      | List of metadata field names to include |
| `collection_name`      | `str`       | Vector store collection name                | `'structured_records'`                  |

Returns:

| Type                                                | Description                                                 |
| --------------------------------------------------- | ----------------------------------------------------------- |
| `tuple[type[Any], VectorStoreCollectionDefinition]` | Tuple of (record_class, definition) for collection creation |

Raises:

| Type         | Description                                             |
| ------------ | ------------------------------------------------------- |
| `ValueError` | If dimensions is invalid (\<=0 or >10000)               |
| `ValueError` | If metadata field name is not a valid Python identifier |

Example

> > > RecordClass, definition = create_structured_record_class( ... dimensions=768, ... metadata_field_names=["title", "category", "price"], ... collection_name="products", ... ) record = RecordClass( ... id="P001", ... content="Product description", ... embedding=[...], ... source_file="products.csv", ... title="Widget Pro", ... category="Electronics", ... price="99.99", ... )

Source code in `src/holodeck/lib/vector_store.py`

```
def create_structured_record_class(
    dimensions: int = 1536,
    metadata_field_names: list[str] | None = None,
    collection_name: str = "structured_records",
) -> tuple[type[Any], VectorStoreCollectionDefinition]:
    """Create a StructuredRecord class and definition for structured data.

    This factory creates a new StructuredRecord dataclass AND a matching
    VectorStoreCollectionDefinition. Both are needed for proper persistence
    of dynamic metadata fields to Semantic Kernel vector stores.

    Unlike DocumentRecord which is for unstructured documents, this is
    designed for structured data (CSV, JSON, JSONL) with user-defined
    metadata fields.

    Args:
        dimensions: Embedding vector dimensions (default: 1536)
        metadata_field_names: List of metadata field names to include
        collection_name: Vector store collection name

    Returns:
        Tuple of (record_class, definition) for collection creation

    Raises:
        ValueError: If dimensions is invalid (<=0 or >10000)
        ValueError: If metadata field name is not a valid Python identifier

    Example:
        >>> RecordClass, definition = create_structured_record_class(
        ...     dimensions=768,
        ...     metadata_field_names=["title", "category", "price"],
        ...     collection_name="products",
        ... )
        >>> record = RecordClass(
        ...     id="P001",
        ...     content="Product description",
        ...     embedding=[...],
        ...     source_file="products.csv",
        ...     title="Widget Pro",
        ...     category="Electronics",
        ...     price="99.99",
        ... )
    """
    if dimensions <= 0 or dimensions > 10000:
        raise ValueError(f"Invalid dimensions: {dimensions}")

    # Validate metadata field names are valid Python identifiers
    if metadata_field_names:
        for field_name in metadata_field_names:
            if not field_name.isidentifier():
                raise ValueError(
                    f"Invalid field name: '{field_name}' "
                    "(must be valid Python identifier)"
                )

    # 1. Create simple dataclass WITHOUT @vectorstoremodel decorator
    # The definition is built separately and passed explicitly to collections
    @dataclass
    class DynamicStructuredRecord:
        """Vector store record for structured data with embeddings.

        Each record represents a row/document from structured data
        (CSV, JSON, JSONL). Compatible with all Semantic Kernel vector
        store backends.

        Attributes:
            id: Unique identifier from the source data's id_field (key)
            content: Concatenated vector field content (full-text indexed)
            embedding: Vector embedding
            source_file: Original source file path (indexed for filtering)
        """

        id: str = ""
        content: str = ""
        embedding: list[float] | None = None
        source_file: str = ""

    # 2. Build VectorStoreField list programmatically
    fields: list[VectorStoreField] = [
        VectorStoreField("key", name="id", type="str"),
        VectorStoreField("data", name="content", type="str", is_full_text_indexed=True),
        VectorStoreField(
            "vector",
            name="embedding",
            type="float",
            dimensions=dimensions,
            distance_function=DistanceFunction.COSINE_SIMILARITY,
        ),
        VectorStoreField("data", name="source_file", type="str", is_indexed=True),
    ]

    # 3. Add dynamic metadata fields to BOTH class and definition
    if metadata_field_names:
        # Add fields to the definition
        for field_name in metadata_field_names:
            fields.append(
                VectorStoreField("data", name=field_name, type="str", is_indexed=True)
            )

        # Store metadata config in a dedicated namespace (explicit and introspectable)
        DynamicStructuredRecord.__metadata_config__ = {  # type: ignore[attr-defined]
            "field_names": metadata_field_names
        }

        # Register the metadata fields in __annotations__ for introspection
        for field_name in metadata_field_names:
            DynamicStructuredRecord.__annotations__[field_name] = str

        # Override __init__ to accept metadata fields as keyword arguments
        original_init = DynamicStructuredRecord.__init__

        def new_init(
            self: Any,
            id: str = "",
            content: str = "",
            embedding: list[float] | None = None,
            source_file: str = "",
            **kwargs: Any,
        ) -> None:
            # Call original with keyword args (not positional) for dataclass compat
            original_init(
                self,
                id=id,
                content=content,
                embedding=embedding,
                source_file=source_file,
            )
            # Store metadata fields as instance attributes
            config = DynamicStructuredRecord.__metadata_config__  # type: ignore[attr-defined]
            for fname in config["field_names"]:
                setattr(self, fname, kwargs.get(fname, ""))

        DynamicStructuredRecord.__init__ = new_init  # type: ignore[method-assign]

    # 4. Build the VectorStoreCollectionDefinition
    # Note: We don't need to_dict/from_dict for individual records -
    # SK will use the field definitions directly for serialization
    definition = VectorStoreCollectionDefinition(
        collection_name=collection_name,
        fields=fields,
    )

    return cast(type[Any], DynamicStructuredRecord), definition
```

## `create_hierarchical_document_record_class(dimensions=1536, tool_name='default')`

Create a HierarchicalDocumentRecord class with specified embedding dimensions.

This factory creates a record class for hierarchical document chunks that preserves document structure, parent-child relationships, cross-references, and definition information. Designed for advanced hybrid search with native full-text indexing.

Parameters:

| Name         | Type  | Description                                                      | Default     |
| ------------ | ----- | ---------------------------------------------------------------- | ----------- |
| `dimensions` | `int` | Embedding vector dimensions (default: 1536)                      | `1536`      |
| `tool_name`  | `str` | Name of the tool for collection namespacing (default: "default") | `'default'` |

Returns:

| Type        | Description                                                              |
| ----------- | ------------------------------------------------------------------------ |
| `type[Any]` | HierarchicalDocumentRecord class configured for the specified dimensions |

Raises:

| Type         | Description                               |
| ------------ | ----------------------------------------- |
| `ValueError` | If dimensions is invalid (\<=0 or >10000) |

Example

> > > RecordClass = create_hierarchical_document_record_class(768, "doc_search") record = RecordClass( ... id="doc_chunk_0", ... source_path="/docs/policy.md", ... chunk_index=0, ... content="Section 1.1 defines the term...", ... embedding=[...], ... parent_chain='["Chapter 1", "Section 1.1"]', ... section_id="1.1", ... chunk_type="definition", ... cross_references='["Section 2.3", "Appendix A"]', ... contextualized_content="This section about X defines...", ... mtime=1706623200.0, ... file_type=".md", ... defined_term="Force Majeure", ... defined_term_normalized="force majeure", ... )

Source code in `src/holodeck/lib/vector_store.py`

```
def create_hierarchical_document_record_class(
    dimensions: int = 1536, tool_name: str = "default"
) -> type[Any]:
    """Create a HierarchicalDocumentRecord class with specified embedding dimensions.

    This factory creates a record class for hierarchical document chunks that
    preserves document structure, parent-child relationships, cross-references,
    and definition information. Designed for advanced hybrid search with
    native full-text indexing.

    Args:
        dimensions: Embedding vector dimensions (default: 1536)
        tool_name: Name of the tool for collection namespacing (default: "default")

    Returns:
        HierarchicalDocumentRecord class configured for the specified dimensions

    Raises:
        ValueError: If dimensions is invalid (<=0 or >10000)

    Example:
        >>> RecordClass = create_hierarchical_document_record_class(768, "doc_search")
        >>> record = RecordClass(
        ...     id="doc_chunk_0",
        ...     source_path="/docs/policy.md",
        ...     chunk_index=0,
        ...     content="Section 1.1 defines the term...",
        ...     embedding=[...],
        ...     parent_chain='["Chapter 1", "Section 1.1"]',
        ...     section_id="1.1",
        ...     chunk_type="definition",
        ...     cross_references='["Section 2.3", "Appendix A"]',
        ...     contextualized_content="This section about X defines...",
        ...     mtime=1706623200.0,
        ...     file_type=".md",
        ...     defined_term="Force Majeure",
        ...     defined_term_normalized="force majeure",
        ... )
    """
    if dimensions <= 0 or dimensions > 10000:
        raise ValueError(f"Invalid dimensions: {dimensions}")

    @vectorstoremodel(collection_name=f"hierarchical_docs_{tool_name}_dim{dimensions}")
    @dataclass
    class DynamicHierarchicalDocumentRecord:  # type: ignore[misc]
        """Vector store record for hierarchical document chunks with structure metadata.

        Each document chunk preserves its position in the document hierarchy,
        cross-references to other sections, and optional definition information.
        Supports native hybrid search via is_full_text_indexed on content field.

        Attributes:
            id: Unique identifier (key field) following format:
                {source_path}_chunk_{chunk_index}
            source_path: Original source file path (indexed for filtering)
            chunk_index: Chunk index within document (0-indexed, indexed)
            section_id: Document section identifier (e.g., "1.2.3") for filtering
            content: Chunk content for semantic search (full-text indexed for hybrid)
            embedding: Vector embedding for semantic similarity search
            parent_chain: JSON-encoded list of ancestor headings
                (e.g., '["Ch1", "Sec1.1"]')
            chunk_type: Classification of content type (content, definition, etc.)
            cross_references: JSON-encoded list of referenced section IDs
            contextualized_content: Content with added context for better retrieval
            mtime: File modification time (Unix timestamp) for change detection
            file_type: Source file extension (.txt, .md, .pdf, etc.)
            defined_term: The term being defined (if chunk_type is definition)
            defined_term_normalized: Lowercase normalized term for
                case-insensitive lookup
        """

        id: Annotated[str, VectorStoreField("key")] = field(
            default_factory=lambda: str(uuid4())
        )
        source_path: Annotated[str, VectorStoreField("data", is_indexed=True)] = field(
            default=""
        )
        chunk_index: Annotated[int, VectorStoreField("data", is_indexed=True)] = field(
            default=0
        )
        section_id: Annotated[str, VectorStoreField("data", is_indexed=True)] = field(
            default=""
        )
        # Display-only fields. Keyword search targets `searchable_text` below
        # (built from a concat of contextualized_content + parent_chain +
        # section_id + defined_term + cross_refs + filename, with implicit
        # boost via repetition — see _build_searchable_text in
        # hierarchical_document_tool.py). SK's hybrid_search only takes one
        # additional_property_name, so we route everything through one field.
        content: Annotated[str, VectorStoreField("data")] = field(default="")
        embedding: Annotated[
            list[float] | None,
            VectorStoreField(
                "vector",
                dimensions=dimensions,
                distance_function=DistanceFunction.COSINE_SIMILARITY,
            ),
        ] = field(default=None)
        parent_chain: Annotated[str, VectorStoreField("data")] = field(default="[]")
        chunk_type: Annotated[str, VectorStoreField("data")] = field(default="content")
        cross_references: Annotated[str, VectorStoreField("data")] = field(default="[]")
        contextualized_content: Annotated[str, VectorStoreField("data")] = field(
            default=""
        )
        searchable_text: Annotated[
            str, VectorStoreField("data", is_full_text_indexed=True)
        ] = field(default="")
        mtime: Annotated[float, VectorStoreField("data")] = field(default=0.0)
        file_type: Annotated[str, VectorStoreField("data")] = field(default="")
        defined_term: Annotated[str, VectorStoreField("data", is_indexed=True)] = field(
            default=""
        )
        defined_term_normalized: Annotated[
            str, VectorStoreField("data", is_indexed=True)
        ] = field(default="")
        subsection_ids: Annotated[str, VectorStoreField("data")] = field(default="[]")
        content_hash: Annotated[str, VectorStoreField("data")] = field(default="")

    return cast(type[Any], DynamicHierarchicalDocumentRecord)
```

## `get_collection_class(provider)`

Lazily import and return the collection class for a provider.

This function imports connector classes on-demand to avoid import errors when optional dependencies are not installed.

Parameters:

| Name       | Type  | Description                | Default    |
| ---------- | ----- | -------------------------- | ---------- |
| `provider` | `str` | Vector store provider name | *required* |

Returns:

| Type        | Description                                     |
| ----------- | ----------------------------------------------- |
| `type[Any]` | The collection class for the specified provider |

Raises:

| Type          | Description                                                 |
| ------------- | ----------------------------------------------------------- |
| `ValueError`  | If provider is not supported                                |
| `ImportError` | If required dependencies for the provider are not installed |

Source code in `src/holodeck/lib/vector_store.py`

```
def get_collection_class(provider: str) -> type[Any]:
    """Lazily import and return the collection class for a provider.

    This function imports connector classes on-demand to avoid import errors
    when optional dependencies are not installed.

    Args:
        provider: Vector store provider name

    Returns:
        The collection class for the specified provider

    Raises:
        ValueError: If provider is not supported
        ImportError: If required dependencies for the provider are not installed
    """
    # Map providers to their import paths and class names
    provider_imports: dict[str, tuple[str, str]] = {
        "postgres": ("semantic_kernel.connectors.postgres", "PostgresCollection"),
        "azure-ai-search": (
            "semantic_kernel.connectors.azure_ai_search",
            "AzureAISearchCollection",
        ),
        "qdrant": ("semantic_kernel.connectors.qdrant", "QdrantCollection"),
        "weaviate": ("semantic_kernel.connectors.weaviate", "WeaviateCollection"),
        "chromadb": ("semantic_kernel.connectors.chroma", "ChromaCollection"),
        "faiss": ("semantic_kernel.connectors.faiss", "FaissCollection"),
        "azure-cosmos-mongo": (
            "semantic_kernel.connectors.azure_cosmos_db",
            "CosmosMongoCollection",
        ),
        "azure-cosmos-nosql": (
            "semantic_kernel.connectors.azure_cosmos_db",
            "CosmosNoSqlCollection",
        ),
        "sql-server": ("semantic_kernel.connectors.sql_server", "SqlServerCollection"),
        "pinecone": ("semantic_kernel.connectors.pinecone", "PineconeCollection"),
        "in-memory": ("semantic_kernel.connectors.in_memory", "InMemoryCollection"),
    }

    if provider not in provider_imports:
        raise ValueError(
            f"Unsupported vector store provider: {provider}. "
            f"Supported providers: {', '.join(sorted(provider_imports.keys()))}"
        )

    module_path, class_name = provider_imports[provider]

    try:
        import importlib

        module = importlib.import_module(module_path)
        return cast(type[Any], getattr(module, class_name))
    except ImportError as e:
        # Provide helpful error message about missing dependencies
        # Use holodeck-ai[extra] format for optional dependencies we provide
        dep_hints: dict[str, str] = {
            "postgres": "holodeck-ai[postgres]",
            "azure-ai-search": "azure-search-documents",
            "qdrant": "holodeck-ai[qdrant]",
            "weaviate": "weaviate-client",
            "chromadb": "holodeck-ai[chromadb]",
            "faiss": "faiss-cpu",
            "azure-cosmos-mongo": "pymongo",
            "azure-cosmos-nosql": "azure-cosmos",
            "sql-server": "pyodbc",
            "pinecone": "holodeck-ai[pinecone]",
        }
        hint = dep_hints.get(provider, "")
        if hint:
            if hint.startswith("holodeck-ai["):
                install_msg = f" Install with: uv add {hint}"
            else:
                install_msg = f" Install with: pip install {hint}"
        else:
            install_msg = ""
        raise ImportError(
            f"Missing dependencies for vector store provider '{provider}'.{install_msg}"
        ) from e
```

## `get_collection_factory(provider, dimensions=1536, record_class=None, definition=None, **connection_kwargs)`

Get a vector store collection factory for the specified provider.

Returns a callable that lazily initializes the appropriate Semantic Kernel collection type based on the provider name and connection parameters.

Parameters:

| Name                  | Type                              | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Default                                                                                                                                                                                      |
| --------------------- | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `provider`            | `str`                             | Vector store provider name. Supported providers: - postgres: PostgreSQL with pgvector extension - azure-ai-search: Azure AI Search (Cognitive Search) - qdrant: Qdrant vector database - weaviate: Weaviate vector database - chromadb: ChromaDB (local or server) - faiss: FAISS (in-memory or file-based) - azure-cosmos-mongo: Azure Cosmos DB (MongoDB API) - azure-cosmos-nosql: Azure Cosmos DB (NoSQL API) - sql-server: SQL Server with vector support - pinecone: Pinecone serverless vector database - in-memory: Simple in-memory storage (development only)                                                                                                                                                                                                                                                                             | *required*                                                                                                                                                                                   |
| `dimensions`          | `int`                             | Embedding vector dimensions (default: 1536). Must be between 1 and 10000.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           | `1536`                                                                                                                                                                                       |
| `definition`          | \`VectorStoreCollectionDefinition | None\`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Optional VectorStoreCollectionDefinition for structured data with dynamic metadata fields. When provided, this definition is passed to the collection constructor for proper field handling. |
| `**connection_kwargs` | `Any`                             | Provider-specific connection parameters. For chromadb provider: - connection_string (str): URL for remote ChromaDB server. Format: "http\[s\]://host[:port]" Examples: "http://localhost:8000", "https://chroma.example.com" - persist_directory (str): Local directory for persistent storage. If provided, creates a PersistentClient instead of HttpClient. - headers (dict[str, str]): HTTP headers for authentication (only used with connection_string) - tenant (str): Tenant name (default: 'default_tenant') - database (str): Database name (default: 'default_database') Note: If neither connection_string nor persist_directory is provided, an ephemeral in-memory client is created. For other providers: Refer to Semantic Kernel documentation for provider-specific connection parameters (e.g., connection_string for postgres). | `{}`                                                                                                                                                                                         |

Returns:

| Type                | Description                                                            |
| ------------------- | ---------------------------------------------------------------------- |
| `Callable[[], Any]` | Callable that returns a Semantic Kernel VectorStoreCollection instance |

Raises:

| Type          | Description                                                 |
| ------------- | ----------------------------------------------------------- |
| `ValueError`  | If provider is not supported or dimensions are invalid      |
| `ImportError` | If required dependencies for the provider are not installed |

Examples:

```
>>> # PostgreSQL with connection string
>>> factory = get_collection_factory(
...     "postgres",
...     dimensions=1536,
...     connection_string="postgresql://user:pass@localhost/db"
... )
>>> async with factory() as collection:
...     await collection.upsert([record])
```

```
>>> # ChromaDB - Connect to remote server
>>> factory = get_collection_factory(
...     "chromadb",
...     dimensions=1536,
...     connection_string="http://localhost:8000"
... )
```

```
>>> # ChromaDB - Connect with authentication headers
>>> factory = get_collection_factory(
...     "chromadb",
...     dimensions=1536,
...     connection_string="https://chroma.example.com",
...     headers={"Authorization": "Bearer token123"}
... )
```

```
>>> # ChromaDB - Persistent local storage
>>> factory = get_collection_factory(
...     "chromadb",
...     dimensions=1536,
...     persist_directory="/var/data/vectors"
... )
```

```
>>> # ChromaDB - Ephemeral in-memory (for testing)
>>> factory = get_collection_factory("chromadb", dimensions=768)
```

```
>>> # In-memory provider (development only)
>>> factory = get_collection_factory("in-memory", dimensions=1536)
```

Source code in `src/holodeck/lib/vector_store.py`

```
def get_collection_factory(
    provider: str,
    dimensions: int = 1536,
    record_class: type[Any] | None = None,
    definition: VectorStoreCollectionDefinition | None = None,
    **connection_kwargs: Any,
) -> Callable[[], Any]:
    """Get a vector store collection factory for the specified provider.

    Returns a callable that lazily initializes the appropriate Semantic Kernel
    collection type based on the provider name and connection parameters.

    Args:
        provider: Vector store provider name. Supported providers:
            - postgres: PostgreSQL with pgvector extension
            - azure-ai-search: Azure AI Search (Cognitive Search)
            - qdrant: Qdrant vector database
            - weaviate: Weaviate vector database
            - chromadb: ChromaDB (local or server)
            - faiss: FAISS (in-memory or file-based)
            - azure-cosmos-mongo: Azure Cosmos DB (MongoDB API)
            - azure-cosmos-nosql: Azure Cosmos DB (NoSQL API)
            - sql-server: SQL Server with vector support
            - pinecone: Pinecone serverless vector database
            - in-memory: Simple in-memory storage (development only)

        dimensions: Embedding vector dimensions (default: 1536).
            Must be between 1 and 10000.

        definition: Optional VectorStoreCollectionDefinition for structured
            data with dynamic metadata fields. When provided, this definition
            is passed to the collection constructor for proper field handling.

        **connection_kwargs: Provider-specific connection parameters.

            For chromadb provider:
                - connection_string (str): URL for remote ChromaDB server.
                  Format: "http[s]://host[:port]"
                  Examples: "http://localhost:8000", "https://chroma.example.com"
                - persist_directory (str): Local directory for persistent storage.
                  If provided, creates a PersistentClient instead of HttpClient.
                - headers (dict[str, str]): HTTP headers for authentication
                  (only used with connection_string)
                - tenant (str): Tenant name (default: 'default_tenant')
                - database (str): Database name (default: 'default_database')

                Note: If neither connection_string nor persist_directory is
                provided, an ephemeral in-memory client is created.

            For other providers:
                Refer to Semantic Kernel documentation for provider-specific
                connection parameters (e.g., connection_string for postgres).

    Returns:
        Callable that returns a Semantic Kernel VectorStoreCollection instance

    Raises:
        ValueError: If provider is not supported or dimensions are invalid
        ImportError: If required dependencies for the provider are not installed

    Examples:
        >>> # PostgreSQL with connection string
        >>> factory = get_collection_factory(
        ...     "postgres",
        ...     dimensions=1536,
        ...     connection_string="postgresql://user:pass@localhost/db"
        ... )
        >>> async with factory() as collection:
        ...     await collection.upsert([record])

        >>> # ChromaDB - Connect to remote server
        >>> factory = get_collection_factory(
        ...     "chromadb",
        ...     dimensions=1536,
        ...     connection_string="http://localhost:8000"
        ... )

        >>> # ChromaDB - Connect with authentication headers
        >>> factory = get_collection_factory(
        ...     "chromadb",
        ...     dimensions=1536,
        ...     connection_string="https://chroma.example.com",
        ...     headers={"Authorization": "Bearer token123"}
        ... )

        >>> # ChromaDB - Persistent local storage
        >>> factory = get_collection_factory(
        ...     "chromadb",
        ...     dimensions=1536,
        ...     persist_directory="/var/data/vectors"
        ... )

        >>> # ChromaDB - Ephemeral in-memory (for testing)
        >>> factory = get_collection_factory("chromadb", dimensions=768)

        >>> # In-memory provider (development only)
        >>> factory = get_collection_factory("in-memory", dimensions=1536)
    """
    supported_providers = [
        "postgres",
        "azure-ai-search",
        "qdrant",
        "weaviate",
        "chromadb",
        "faiss",
        "azure-cosmos-mongo",
        "azure-cosmos-nosql",
        "sql-server",
        "pinecone",
        "in-memory",
    ]

    # Validate dimensions
    if dimensions <= 0 or dimensions > 10000:
        raise ValueError(f"Invalid dimensions: {dimensions}")

    if provider not in supported_providers:
        raise ValueError(
            f"Unsupported vector store provider: {provider}. "
            f"Supported providers: {', '.join(sorted(supported_providers))}"
        )

    # Use provided record_class or create default DocumentRecord
    if record_class is None:
        record_class = create_document_record_class(dimensions)

    # Pre-process provider-specific kwargs to avoid mutating original dict in factory
    # Each provider may need connection_string parsed into specific parameters

    # ChromaDB handling
    if provider == "chromadb":
        chromadb_connection_string = connection_kwargs.pop("connection_string", None)
        chromadb_persist_directory = connection_kwargs.pop("persist_directory", None)
        chromadb_extra_kwargs = connection_kwargs.copy()
    else:
        chromadb_connection_string = None
        chromadb_persist_directory = None
        chromadb_extra_kwargs = {}

    # Qdrant handling - parse connection_string into Qdrant-specific params
    if provider == "qdrant":
        qdrant_params: QdrantConnectionParams = {}
        if "connection_string" in connection_kwargs:
            qdrant_params = parse_qdrant_connection_string(
                connection_kwargs.pop("connection_string")
            )
        # Merge any explicit kwargs (they override parsed values)
        qdrant_params.update(connection_kwargs)  # type: ignore[typeddict-item]
    else:
        qdrant_params = {}

    # Pinecone handling - parse connection_string or use api_key directly
    if provider == "pinecone":
        pinecone_params: PineconeConnectionParams = {}
        if "connection_string" in connection_kwargs:
            pinecone_params = parse_pinecone_connection_string(
                connection_kwargs.pop("connection_string")
            )
        # Merge any explicit kwargs (api_key, namespace, use_grpc)
        if "api_key" in connection_kwargs:
            pinecone_params["api_key"] = connection_kwargs.pop("api_key")
        if "namespace" in connection_kwargs:
            pinecone_params["namespace"] = connection_kwargs.pop("namespace")
        if "use_grpc" in connection_kwargs:
            pinecone_params["use_grpc"] = connection_kwargs.pop("use_grpc")
        pinecone_extra_kwargs = connection_kwargs.copy()
    else:
        pinecone_params = {}
        pinecone_extra_kwargs = {}

    # PostgreSQL handling - connection_string is passed directly to SK
    # SK's PostgresSettings handles parsing internally
    if provider == "postgres":
        postgres_params: PostgresConnectionParams = {}
        if "connection_string" in connection_kwargs:
            # Pass connection_string to SK which will parse it via PostgresSettings
            postgres_params["connection_string"] = connection_kwargs.pop(
                "connection_string"
            )
        # Handle db_schema separately
        if "db_schema" in connection_kwargs:
            postgres_params["db_schema"] = connection_kwargs.pop("db_schema")
        postgres_extra_kwargs = connection_kwargs.copy()
    else:
        postgres_params = {}
        postgres_extra_kwargs = {}

    def factory() -> Any:
        """Return async context manager for the collection."""
        # Lazy import at factory call time
        collection_class = get_collection_class(provider)

        # Build base kwargs with optional definition for structured data
        base_kwargs: dict[str, Any] = {"record_type": record_class}
        if definition is not None:
            base_kwargs["definition"] = definition

        # ChromaDB requires special handling for connection_string
        if provider == "chromadb":
            # Create the appropriate client
            client = create_chromadb_client(
                connection_string=chromadb_connection_string,
                persist_directory=chromadb_persist_directory,
                **chromadb_extra_kwargs,
            )

            # Pass the pre-configured client to ChromaCollection
            return collection_class[str, record_class](
                client=client,
                **base_kwargs,
            )

        # Qdrant - construct the AsyncQdrantClient ourselves and pass it as
        # `client=` so SK marks it managed_client=False. Otherwise SK calls
        # `qdrant_client.close()` on every `async with collection: …` exit,
        # and the second `async with` on the same collection fails with
        # "Cannot send a request, as the client has been closed." Multiple
        # enter/exit cycles per tool are normal (load → needs_reingest →
        # store → search), so we keep client lifetime tied to the tool, not
        # to the context manager.
        if provider == "qdrant":
            from qdrant_client import AsyncQdrantClient

            qdrant_client = AsyncQdrantClient(**qdrant_params)  # type: ignore[arg-type]
            return collection_class[str, record_class](
                client=qdrant_client,
                **base_kwargs,
            )

        # Pinecone - pass parsed parameters to PineconeCollection
        if provider == "pinecone":
            return collection_class[str, record_class](
                **base_kwargs,
                **pinecone_params,
                **pinecone_extra_kwargs,
            )

        # PostgreSQL - SK handles connection_string parsing via PostgresSettings
        if provider == "postgres":
            # PostgresCollection accepts settings or individual connection params
            # We pass connection_string which PostgresSettings will parse
            kwargs_for_postgres: dict[str, Any] = base_kwargs.copy()
            conn_str = postgres_params.get("connection_string")
            if conn_str:
                # Create PostgresSettings with connection_string
                # SK will handle the parsing internally
                from pydantic import SecretStr
                from semantic_kernel.connectors.postgres import PostgresSettings

                settings = PostgresSettings(connection_string=SecretStr(conn_str))
                kwargs_for_postgres["settings"] = settings
            db_schema = postgres_params.get("db_schema")
            if db_schema:
                kwargs_for_postgres["db_schema"] = db_schema
            kwargs_for_postgres.update(postgres_extra_kwargs)
            return collection_class[str, record_class](**kwargs_for_postgres)

        # Default handling for other providers
        return collection_class[str, record_class](
            **base_kwargs,
            **connection_kwargs,
        )

    return factory
```

## `parse_chromadb_connection_string(connection_string)`

Parse a ChromaDB connection string into connection parameters.

Supports URL format: `http[s]://[host][:port][/path]`

The connection string follows standard URL conventions:

- Scheme (http/https) determines SSL setting
- Host defaults to 'localhost' if not specified
- Port defaults to 8000 for HTTP, 443 for HTTPS

Parameters:

| Name                | Type  | Description                                     | Default    |
| ------------------- | ----- | ----------------------------------------------- | ---------- |
| `connection_string` | `str` | URL-style connection string for ChromaDB server | *required* |

Returns:

| Type                     | Description                                            |
| ------------------------ | ------------------------------------------------------ |
| `ChromaConnectionParams` | ChromaConnectionParams with host, port, and ssl values |

Raises:

| Type         | Description                                              |
| ------------ | -------------------------------------------------------- |
| `ValueError` | If connection string is empty or uses unsupported scheme |

Examples:

```
>>> parse_chromadb_connection_string("http://localhost:8000")
{'host': 'localhost', 'port': 8000, 'ssl': False}
```

```
>>> parse_chromadb_connection_string("https://chroma.example.com")
{'host': 'chroma.example.com', 'port': 443, 'ssl': True}
```

```
>>> parse_chromadb_connection_string("http://localhost")
{'host': 'localhost', 'port': 8000, 'ssl': False}
```

```
>>> parse_chromadb_connection_string("https://chroma.internal:9000")
{'host': 'chroma.internal', 'port': 9000, 'ssl': True}
```

Source code in `src/holodeck/lib/vector_store.py`

```
def parse_chromadb_connection_string(connection_string: str) -> ChromaConnectionParams:
    """Parse a ChromaDB connection string into connection parameters.

    Supports URL format: `http[s]://[host][:port][/path]`

    The connection string follows standard URL conventions:
    - Scheme (http/https) determines SSL setting
    - Host defaults to 'localhost' if not specified
    - Port defaults to 8000 for HTTP, 443 for HTTPS

    Args:
        connection_string: URL-style connection string for ChromaDB server

    Returns:
        ChromaConnectionParams with host, port, and ssl values

    Raises:
        ValueError: If connection string is empty or uses unsupported scheme

    Examples:
        >>> parse_chromadb_connection_string("http://localhost:8000")
        {'host': 'localhost', 'port': 8000, 'ssl': False}

        >>> parse_chromadb_connection_string("https://chroma.example.com")
        {'host': 'chroma.example.com', 'port': 443, 'ssl': True}

        >>> parse_chromadb_connection_string("http://localhost")
        {'host': 'localhost', 'port': 8000, 'ssl': False}

        >>> parse_chromadb_connection_string("https://chroma.internal:9000")
        {'host': 'chroma.internal', 'port': 9000, 'ssl': True}
    """
    if not connection_string:
        raise ValueError("Connection string cannot be empty")

    parsed = urlparse(connection_string)

    if parsed.scheme not in ("http", "https"):
        raise ValueError(
            f"Invalid scheme '{parsed.scheme}'. ChromaDB connection string must use "
            "http:// or https:// scheme"
        )

    ssl = parsed.scheme == "https"
    host = parsed.hostname or "localhost"

    # Default ports based on scheme
    port = parsed.port or (443 if ssl else 8000)

    params: ChromaConnectionParams = {
        "host": host,
        "port": port,
        "ssl": ssl,
    }

    return params
```

## `parse_qdrant_connection_string(connection_string)`

Parse a Qdrant connection string into connection parameters.

Supports multiple formats for flexibility:

- Standard URL: https://host:port or http://localhost:6333
- With API key in userinfo: https://api_key@host:port
- gRPC preference: qdrant+grpc://host:port
- In-memory: :memory:
- Local path: /path/to/qdrant/data or file:///path/to/data

The connection string is parsed and mapped to QdrantCollection parameters.

Parameters:

| Name                | Type  | Description                                       | Default    |
| ------------------- | ----- | ------------------------------------------------- | ---------- |
| `connection_string` | `str` | Connection string in one of the supported formats | *required* |

Returns:

| Type                     | Description                                        |
| ------------------------ | -------------------------------------------------- |
| `QdrantConnectionParams` | QdrantConnectionParams with appropriate fields set |

Raises:

| Type         | Description                                              |
| ------------ | -------------------------------------------------------- |
| `ValueError` | If connection string is empty or uses unsupported scheme |

Examples:

```
>>> parse_qdrant_connection_string("https://qdrant.example.com:6333")
{'url': 'https://qdrant.example.com:6333'}
```

```
>>> parse_qdrant_connection_string("http://localhost:6333")
{'host': 'localhost', 'port': 6333}
```

```
>>> parse_qdrant_connection_string(":memory:")
{'location': ':memory:'}
```

```
>>> parse_qdrant_connection_string("qdrant+grpc://localhost:6334")
{'host': 'localhost', 'grpc_port': 6334, 'prefer_grpc': True}
```

Source code in `src/holodeck/lib/vector_store.py`

```
def parse_qdrant_connection_string(connection_string: str) -> QdrantConnectionParams:
    """Parse a Qdrant connection string into connection parameters.

    Supports multiple formats for flexibility:
    - Standard URL: https://host:port or http://localhost:6333
    - With API key in userinfo: https://api_key@host:port
    - gRPC preference: qdrant+grpc://host:port
    - In-memory: :memory:
    - Local path: /path/to/qdrant/data or file:///path/to/data

    The connection string is parsed and mapped to QdrantCollection parameters.

    Args:
        connection_string: Connection string in one of the supported formats

    Returns:
        QdrantConnectionParams with appropriate fields set

    Raises:
        ValueError: If connection string is empty or uses unsupported scheme

    Examples:
        >>> parse_qdrant_connection_string("https://qdrant.example.com:6333")
        {'url': 'https://qdrant.example.com:6333'}

        >>> parse_qdrant_connection_string("http://localhost:6333")
        {'host': 'localhost', 'port': 6333}

        >>> parse_qdrant_connection_string(":memory:")
        {'location': ':memory:'}

        >>> parse_qdrant_connection_string("qdrant+grpc://localhost:6334")
        {'host': 'localhost', 'grpc_port': 6334, 'prefer_grpc': True}
    """
    if not connection_string:
        raise ValueError("Connection string cannot be empty")

    # Handle special in-memory location
    if connection_string == ":memory:":
        return {"location": ":memory:"}

    # Handle local file path (for persistent local storage)
    if connection_string.startswith("/") or connection_string.startswith("file://"):
        path = connection_string.replace("file://", "")
        return {"path": path}

    parsed = urlparse(connection_string)

    # Handle gRPC preference via scheme
    prefer_grpc = parsed.scheme in ("qdrant+grpc", "grpc")

    # Normalize scheme for URL construction
    if parsed.scheme in ("qdrant", "qdrant+grpc", "grpc"):
        # Convert custom schemes to http/https
        actual_scheme = "https" if parsed.port == 443 else "http"
    elif parsed.scheme in ("http", "https"):
        actual_scheme = parsed.scheme
    else:
        raise ValueError(
            f"Invalid scheme '{parsed.scheme}'. Qdrant connection string must use "
            "http://, https://, qdrant://, or qdrant+grpc:// scheme"
        )

    host = parsed.hostname or "localhost"

    # Default ports based on transport
    if prefer_grpc:
        grpc_port = parsed.port or 6334
        params: QdrantConnectionParams = {
            "host": host,
            "grpc_port": grpc_port,
            "prefer_grpc": True,
        }
    else:
        port = parsed.port or 6333
        # For remote servers, pass full URL; for localhost use host/port
        if host == "localhost" or host == "127.0.0.1":
            params = {
                "host": host,
                "port": port,
            }
        else:
            # Use full URL for remote servers
            url = f"{actual_scheme}://{host}"
            if parsed.port:
                url += f":{parsed.port}"
            params = {"url": url}

    # Extract API key from userinfo if present
    if parsed.username:
        params["api_key"] = parsed.username

    return params
```

## `parse_pinecone_connection_string(connection_string)`

Parse a Pinecone connection string into connection parameters.

Pinecone primarily uses API key authentication. The connection string can be the API key directly or a URL-like format for consistency.

Supported formats:

- Direct API key: "pc-abc123..." (starts with 'pc-')
- URL format: pinecone://api_key or pinecone://api_key@namespace

Parameters:

| Name                | Type  | Description                    | Default    |
| ------------------- | ----- | ------------------------------ | ---------- |
| `connection_string` | `str` | Connection string with API key | *required* |

Returns:

| Type                       | Description                                                  |
| -------------------------- | ------------------------------------------------------------ |
| `PineconeConnectionParams` | PineconeConnectionParams with api_key and optional namespace |

Raises:

| Type         | Description                   |
| ------------ | ----------------------------- |
| `ValueError` | If connection string is empty |

Examples:

```
>>> parse_pinecone_connection_string("pc-abc123def456")
{'api_key': 'pc-abc123def456'}
```

```
>>> parse_pinecone_connection_string("pinecone://pc-abc123")
{'api_key': 'pc-abc123'}
```

```
>>> parse_pinecone_connection_string("pinecone://pc-abc123@my-namespace")
{'api_key': 'pc-abc123', 'namespace': 'my-namespace'}
```

Source code in `src/holodeck/lib/vector_store.py`

```
def parse_pinecone_connection_string(
    connection_string: str,
) -> PineconeConnectionParams:
    """Parse a Pinecone connection string into connection parameters.

    Pinecone primarily uses API key authentication. The connection string
    can be the API key directly or a URL-like format for consistency.

    Supported formats:
    - Direct API key: "pc-abc123..." (starts with 'pc-')
    - URL format: pinecone://api_key or pinecone://api_key@namespace

    Args:
        connection_string: Connection string with API key

    Returns:
        PineconeConnectionParams with api_key and optional namespace

    Raises:
        ValueError: If connection string is empty

    Examples:
        >>> parse_pinecone_connection_string("pc-abc123def456")
        {'api_key': 'pc-abc123def456'}

        >>> parse_pinecone_connection_string("pinecone://pc-abc123")
        {'api_key': 'pc-abc123'}

        >>> parse_pinecone_connection_string("pinecone://pc-abc123@my-namespace")
        {'api_key': 'pc-abc123', 'namespace': 'my-namespace'}
    """
    if not connection_string:
        raise ValueError("Connection string cannot be empty")

    # Direct API key format (Pinecone keys typically start with 'pc-')
    if connection_string.startswith("pc-") or not connection_string.startswith(
        "pinecone://"
    ):
        return {"api_key": connection_string}

    # URL-like format: pinecone://api_key[@namespace]
    parsed = urlparse(connection_string)

    if parsed.scheme != "pinecone":
        raise ValueError(
            f"Invalid scheme '{parsed.scheme}'. "
            "Use 'pinecone://' or provide API key directly"
        )

    params: PineconeConnectionParams = {}

    # API key can be in username position or hostname
    if parsed.username:
        params["api_key"] = parsed.username
        if parsed.hostname:
            params["namespace"] = parsed.hostname
    elif parsed.hostname:
        params["api_key"] = parsed.hostname

    # Check for namespace in path
    if parsed.path and parsed.path.startswith("/"):
        namespace = parsed.path[1:]  # Remove leading slash
        if namespace:
            params["namespace"] = namespace

    return params
```

______________________________________________________________________

## Logging

Centralized logging configuration with support for console and file handlers, environment variable configuration, log rotation, and structured logging patterns.

### Logging Configuration

## `setup_logging(level=None, log_file=None, log_format=None, verbose=False, quiet=False)`

Configure logging for HoloDeck application.

This function sets up the root logger with appropriate handlers and formatters. It respects environment variables and command-line flags for configuration. It also configures third-party library loggers to respect the quiet flag.

Parameters:

| Name         | Type   | Description                                                                                                              | Default |
| ------------ | ------ | ------------------------------------------------------------------------------------------------------------------------ | ------- |
| `level`      | `str`  | Log level (DEBUG, INFO, WARNING, ERROR, CRITICAL). If not provided, uses HOLODECK_LOG_LEVEL env var or defaults to INFO. | `None`  |
| `log_file`   | `str`  | Path to log file. If not provided, uses HOLODECK_LOG_FILE env var. If neither is set, only console logging is used.      | `None`  |
| `log_format` | `str`  | Log format string. If not provided, uses HOLODECK_LOG_FORMAT env var or default format.                                  | `None`  |
| `verbose`    | `bool` | If True, sets log level to DEBUG. Overrides level parameter.                                                             | `False` |
| `quiet`      | `bool` | If True, sets log level to ERROR. Overrides verbose and level.                                                           | `False` |

Returns:

| Type   | Description |
| ------ | ----------- |
| `None` | None        |

Example

> > > setup_logging(verbose=True) # Enable DEBUG logging setup_logging(quiet=True) # Only show ERROR and above setup_logging(log_file="/var/log/holodeck.log") # Enable file logging

Source code in `src/holodeck/lib/logging_config.py`

```
def setup_logging(
    level: str | None = None,
    log_file: str | None = None,
    log_format: str | None = None,
    verbose: bool = False,
    quiet: bool = False,
) -> None:
    """
    Configure logging for HoloDeck application.

    This function sets up the root logger with appropriate handlers and formatters.
    It respects environment variables and command-line flags for configuration.
    It also configures third-party library loggers to respect the quiet flag.

    Parameters:
        level (str, optional): Log level (DEBUG, INFO, WARNING, ERROR, CRITICAL).
            If not provided, uses HOLODECK_LOG_LEVEL env var or defaults to INFO.
        log_file (str, optional): Path to log file. If not provided, uses
            HOLODECK_LOG_FILE env var. If neither is set, only console logging is used.
        log_format (str, optional): Log format string. If not provided, uses
            HOLODECK_LOG_FORMAT env var or default format.
        verbose (bool): If True, sets log level to DEBUG. Overrides level parameter.
        quiet (bool): If True, sets log level to ERROR. Overrides verbose and level.

    Returns:
        None

    Example:
        >>> setup_logging(verbose=True)  # Enable DEBUG logging
        >>> setup_logging(quiet=True)    # Only show ERROR and above
        >>> setup_logging(log_file="/var/log/holodeck.log")  # Enable file logging
    """
    # Determine log level based on flags and configuration
    if quiet:
        log_level = logging.WARNING
    elif verbose:
        log_level = logging.DEBUG
    elif level:
        log_level = getattr(logging, level.upper(), logging.INFO)
    else:
        env_level = os.getenv(ENV_LOG_LEVEL, "INFO")
        log_level = getattr(logging, env_level.upper(), logging.INFO)

    # Determine log format
    if log_format is None:
        log_format = os.getenv(ENV_LOG_FORMAT, DEFAULT_LOG_FORMAT)

    # Determine log file path
    if log_file is None:
        log_file = os.getenv(ENV_LOG_FILE)

    # Configure root logger
    root_logger = logging.getLogger()
    root_logger.setLevel(log_level)

    # Remove existing handlers to avoid duplicates
    root_logger.handlers.clear()

    # Create formatter
    formatter = logging.Formatter(log_format, datefmt=DEFAULT_DATE_FORMAT)

    # Add console handler. Use stderr so log lines never interleave with
    # stdout writers that own cursor positioning (e.g. holodeck.chat.LiveComposer
    # pins a panel below the streaming cursor and breaks if anything else writes
    # to stdout between paints).
    console_handler = logging.StreamHandler(sys.stderr)
    console_handler.setLevel(log_level)
    console_handler.setFormatter(formatter)
    root_logger.addHandler(console_handler)

    # Add file handler if log file is specified
    if log_file:
        _add_file_handler(root_logger, log_file, log_level, formatter)

    # Configure all existing loggers to respect the log level
    # This ensures quiet mode works even for loggers created during imports
    _configure_all_loggers(log_level)

    # Log initial setup message at DEBUG level
    logger = logging.getLogger(__name__)
    logger.debug(
        f"Logging configured: level={logging.getLevelName(log_level)}, "
        f"file={'enabled' if log_file else 'disabled'}"
    )
```

## `get_logger(name)`

Get a logger instance for the specified module.

This is a convenience wrapper around logging.getLogger() that ensures consistent logger naming across the application.

Parameters:

| Name   | Type  | Description                                               | Default    |
| ------ | ----- | --------------------------------------------------------- | ---------- |
| `name` | `str` | Name of the logger, typically name of the calling module. | *required* |

Returns:

| Type     | Description                                             |
| -------- | ------------------------------------------------------- |
| `Logger` | logging.Logger: Logger instance for the specified name. |

Example

> > > logger = get_logger(**name**) logger.info("Processing started")

Source code in `src/holodeck/lib/logging_config.py`

```
def get_logger(name: str) -> logging.Logger:
    """
    Get a logger instance for the specified module.

    This is a convenience wrapper around logging.getLogger() that ensures
    consistent logger naming across the application.

    Parameters:
        name (str): Name of the logger, typically __name__ of the calling module.

    Returns:
        logging.Logger: Logger instance for the specified name.

    Example:
        >>> logger = get_logger(__name__)
        >>> logger.info("Processing started")
    """
    return logging.getLogger(name)
```

## `set_log_level(level)`

Dynamically change the log level for all loggers.

Parameters:

| Name    | Type  | Description                                            | Default    |
| ------- | ----- | ------------------------------------------------------ | ---------- |
| `level` | `str` | New log level (DEBUG, INFO, WARNING, ERROR, CRITICAL). | *required* |

Returns:

| Type   | Description |
| ------ | ----------- |
| `None` | None        |

Example

> > > set_log_level("DEBUG") # Enable debug logging

Source code in `src/holodeck/lib/logging_config.py`

```
def set_log_level(level: str) -> None:
    """
    Dynamically change the log level for all loggers.

    Parameters:
        level (str): New log level (DEBUG, INFO, WARNING, ERROR, CRITICAL).

    Returns:
        None

    Example:
        >>> set_log_level("DEBUG")  # Enable debug logging
    """
    log_level = getattr(logging, level.upper(), logging.INFO)
    root_logger = logging.getLogger()
    root_logger.setLevel(log_level)

    # Update all handlers
    for handler in root_logger.handlers:
        handler.setLevel(log_level)

    # Also update all existing loggers
    _configure_all_loggers(log_level)

    logger = get_logger(__name__)
    logger.debug(f"Log level changed to {logging.getLevelName(log_level)}")
```

## `configure_third_party_loggers(log_level)`

Configure known third-party loggers to respect the specified log level.

This suppresses noisy INFO logs from libraries like httpx, chromadb, etc. Can be called from both traditional logging and OTel logging setup.

Setting level on parent logger (e.g., "chromadb") also affects child loggers (e.g., "chromadb.telemetry.product.posthog").

Parameters:

| Name        | Type  | Description                                        | Default    |
| ----------- | ----- | -------------------------------------------------- | ---------- |
| `log_level` | `int` | The logging level to apply (e.g., logging.WARNING) | *required* |

Source code in `src/holodeck/lib/logging_config.py`

```
def configure_third_party_loggers(log_level: int) -> None:
    """Configure known third-party loggers to respect the specified log level.

    This suppresses noisy INFO logs from libraries like httpx, chromadb, etc.
    Can be called from both traditional logging and OTel logging setup.

    Setting level on parent logger (e.g., "chromadb") also affects child
    loggers (e.g., "chromadb.telemetry.product.posthog").

    Args:
        log_level: The logging level to apply (e.g., logging.WARNING)
    """
    for logger_name in THIRD_PARTY_LOGGERS:
        third_party_logger = logging.getLogger(logger_name)
        third_party_logger.setLevel(log_level)
        third_party_logger.handlers.clear()
        third_party_logger.propagate = True
```

### Logging Utilities

## `LogTimer(logger, level=logging.INFO)`

Timer utility for logging operation durations.

This class provides a simple way to measure and log operation durations.

Example

> > > logger = logging.getLogger(**name**) timer = LogTimer(logger) timer.start("Processing batch")
> > >
> > > ### Do work...
> > >
> > > timer.stop() # Logs: "Processing batch completed in X.XXs"

Initialize the timer.

Parameters:

| Name     | Type     | Description                        | Default    |
| -------- | -------- | ---------------------------------- | ---------- |
| `logger` | `Logger` | Logger to use for timing messages. | *required* |
| `level`  | `int`    | Log level to use (default: INFO).  | `INFO`     |

Source code in `src/holodeck/lib/logging_utils.py`

```
def __init__(self, logger: logging.Logger, level: int = logging.INFO):
    """
    Initialize the timer.

    Parameters:
        logger (logging.Logger): Logger to use for timing messages.
        level (int): Log level to use (default: INFO).
    """
    self.logger = logger
    self.level = level
    self.start_time: float | None = None
    self.operation: str | None = None
```

### `start(operation)`

Start timing an operation.

Parameters:

| Name        | Type  | Description                        | Default    |
| ----------- | ----- | ---------------------------------- | ---------- |
| `operation` | `str` | Name/description of the operation. | *required* |

Source code in `src/holodeck/lib/logging_utils.py`

```
def start(self, operation: str) -> None:
    """
    Start timing an operation.

    Parameters:
        operation (str): Name/description of the operation.
    """
    self.operation = operation
    self.start_time = time.time()
    self.logger.log(self.level, f"{operation} started")
```

### `stop(context=None)`

Stop timing and log the elapsed time.

Parameters:

| Name      | Type   | Description                           | Default |
| --------- | ------ | ------------------------------------- | ------- |
| `context` | `dict` | Additional context to include in log. | `None`  |

Returns:

| Name    | Type    | Description              |
| ------- | ------- | ------------------------ |
| `float` | `float` | Elapsed time in seconds. |

Raises:

| Type         | Description                      |
| ------------ | -------------------------------- |
| `ValueError` | If start() was not called first. |

Source code in `src/holodeck/lib/logging_utils.py`

```
def stop(self, context: dict[str, Any] | None = None) -> float:
    """
    Stop timing and log the elapsed time.

    Parameters:
        context (dict, optional): Additional context to include in log.

    Returns:
        float: Elapsed time in seconds.

    Raises:
        ValueError: If start() was not called first.
    """
    if self.start_time is None or self.operation is None:
        raise ValueError("Timer not started. Call start() first.")

    elapsed = time.time() - self.start_time
    context_str = _format_context(context) if context else ""
    self.logger.log(
        self.level,
        f"{self.operation} completed in {elapsed:.2f}s{context_str}",
    )

    self.start_time = None
    self.operation = None
    return elapsed
```

## `log_operation(logger, operation, level=logging.INFO, context=None)`

Context manager for logging operation start, completion, and timing.

Logs the start of an operation, then logs its completion with elapsed time. If an exception occurs, logs the error with the operation context.

Parameters:

| Name        | Type     | Description                            | Default    |
| ----------- | -------- | -------------------------------------- | ---------- |
| `logger`    | `Logger` | Logger to use for logging.             | *required* |
| `operation` | `str`    | Name/description of the operation.     | *required* |
| `level`     | `int`    | Log level to use (default: INFO).      | `INFO`     |
| `context`   | `dict`   | Additional context to include in logs. | `None`     |

Yields:

| Type   | Description |
| ------ | ----------- |
| `None` | None        |

Example

> > > logger = logging.getLogger(**name**) with log_operation(logger, "Processing file", context={"file": "test.txt"}): ... # Do work here ... process_file("test.txt")

Source code in `src/holodeck/lib/logging_utils.py`

```
@contextmanager
def log_operation(
    logger: logging.Logger,
    operation: str,
    level: int = logging.INFO,
    context: dict[str, Any] | None = None,
) -> Iterator[None]:
    """
    Context manager for logging operation start, completion, and timing.

    Logs the start of an operation, then logs its completion with elapsed time.
    If an exception occurs, logs the error with the operation context.

    Parameters:
        logger (logging.Logger): Logger to use for logging.
        operation (str): Name/description of the operation.
        level (int): Log level to use (default: INFO).
        context (dict, optional): Additional context to include in logs.

    Yields:
        None

    Example:
        >>> logger = logging.getLogger(__name__)
        >>> with log_operation(logger, "Processing file", context={"file": "test.txt"}):
        ...     # Do work here
        ...     process_file("test.txt")
    """
    context_str = _format_context(context) if context else ""
    start_time = time.time()

    logger.log(level, f"{operation} started{context_str}")

    try:
        yield
        elapsed = time.time() - start_time
        logger.log(level, f"{operation} completed in {elapsed:.2f}s{context_str}")
    except Exception as e:
        elapsed = time.time() - start_time
        logger.error(
            f"{operation} failed after {elapsed:.2f}s{context_str}: {e}",
            exc_info=True,
        )
        raise
```

## `log_context(logger, **kwargs)`

Context manager for adding structured context to log messages.

This is useful for adding contextual information that should be included in all log messages within a specific scope.

Parameters:

| Name       | Type     | Description                            | Default    |
| ---------- | -------- | -------------------------------------- | ---------- |
| `logger`   | `Logger` | Logger to use.                         | *required* |
| `**kwargs` | `Any`    | Key-value pairs to add to log context. | `{}`       |

Yields:

| Type   | Description |
| ------ | ----------- |
| `None` | None        |

Example

> > > logger = logging.getLogger(**name**) with log_context(logger, test_id="test-001", attempt=1): ... logger.info("Processing test") # Includes test_id and attempt

Source code in `src/holodeck/lib/logging_utils.py`

```
@contextmanager
def log_context(
    logger: logging.Logger,
    **kwargs: Any,
) -> Iterator[None]:
    """
    Context manager for adding structured context to log messages.

    This is useful for adding contextual information that should be included
    in all log messages within a specific scope.

    Parameters:
        logger (logging.Logger): Logger to use.
        **kwargs: Key-value pairs to add to log context.

    Yields:
        None

    Example:
        >>> logger = logging.getLogger(__name__)
        >>> with log_context(logger, test_id="test-001", attempt=1):
        ...     logger.info("Processing test")  # Includes test_id and attempt
    """
    # Store original context
    original_extra = getattr(logger, "_holodeck_context", {})

    # Add new context
    new_context = {**original_extra, **kwargs}
    logger._holodeck_context = new_context  # type: ignore

    try:
        yield
    finally:
        # Restore original context
        logger._holodeck_context = original_extra  # type: ignore
```

## `log_with_context(logger, level, message, **context)`

Log a message with structured context.

Parameters:

| Name        | Type     | Description                         | Default    |
| ----------- | -------- | ----------------------------------- | ---------- |
| `logger`    | `Logger` | Logger to use.                      | *required* |
| `level`     | `int`    | Log level.                          | *required* |
| `message`   | `str`    | Log message.                        | *required* |
| `**context` | `Any`    | Additional context key-value pairs. | `{}`       |

Example

> > > logger = logging.getLogger(**name**) log_with_context( ... logger, ... logging.INFO, ... "Test passed", ... test_id="test-001", ... duration=1.23 ... )

Source code in `src/holodeck/lib/logging_utils.py`

```
def log_with_context(
    logger: logging.Logger,
    level: int,
    message: str,
    **context: Any,
) -> None:
    """
    Log a message with structured context.

    Parameters:
        logger (logging.Logger): Logger to use.
        level (int): Log level.
        message (str): Log message.
        **context: Additional context key-value pairs.

    Example:
        >>> logger = logging.getLogger(__name__)
        >>> log_with_context(
        ...     logger,
        ...     logging.INFO,
        ...     "Test passed",
        ...     test_id="test-001",
        ...     duration=1.23
        ... )
    """
    context_str = _format_context(context) if context else ""
    logger.log(level, f"{message}{context_str}")
```

## `log_exception(logger, message, exc, level=logging.ERROR, context=None)`

Log an exception with context and stack trace.

Parameters:

| Name      | Type        | Description                           | Default    |
| --------- | ----------- | ------------------------------------- | ---------- |
| `logger`  | `Logger`    | Logger to use.                        | *required* |
| `message` | `str`       | Error message describing what failed. | *required* |
| `exc`     | `Exception` | The exception that occurred.          | *required* |
| `level`   | `int`       | Log level (default: ERROR).           | `ERROR`    |
| `context` | `dict`      | Additional context information.       | `None`     |

Example

> > > logger = logging.getLogger(**name**) try: ... risky_operation() ... except Exception as e: ... log_exception(logger, "Operation failed", e, context={"id": "123"})

Source code in `src/holodeck/lib/logging_utils.py`

```
def log_exception(
    logger: logging.Logger,
    message: str,
    exc: Exception,
    level: int = logging.ERROR,
    context: dict[str, Any] | None = None,
) -> None:
    """
    Log an exception with context and stack trace.

    Parameters:
        logger (logging.Logger): Logger to use.
        message (str): Error message describing what failed.
        exc (Exception): The exception that occurred.
        level (int): Log level (default: ERROR).
        context (dict, optional): Additional context information.

    Example:
        >>> logger = logging.getLogger(__name__)
        >>> try:
        ...     risky_operation()
        ... except Exception as e:
        ...     log_exception(logger, "Operation failed", e, context={"id": "123"})
    """
    context_str = _format_context(context) if context else ""
    logger.log(
        level,
        f"{message}{context_str}: {type(exc).__name__}: {exc}",
        exc_info=True,
    )
```

## `log_retry(logger, operation, attempt, max_attempts, delay, error=None)`

Log a retry attempt with structured context.

Parameters:

| Name           | Type        | Description                          | Default    |
| -------------- | ----------- | ------------------------------------ | ---------- |
| `logger`       | `Logger`    | Logger to use.                       | *required* |
| `operation`    | `str`       | Name of the operation being retried. | *required* |
| `attempt`      | `int`       | Current attempt number.              | *required* |
| `max_attempts` | `int`       | Maximum number of attempts.          | *required* |
| `delay`        | `float`     | Delay before next retry in seconds.  | *required* |
| `error`        | `Exception` | The error that caused the retry.     | `None`     |

Example

> > > logger = logging.getLogger(**name**) log_retry(logger, "API call", attempt=2, max_attempts=3, delay=5.0)

Source code in `src/holodeck/lib/logging_utils.py`

```
def log_retry(
    logger: logging.Logger,
    operation: str,
    attempt: int,
    max_attempts: int,
    delay: float,
    error: Exception | None = None,
) -> None:
    """
    Log a retry attempt with structured context.

    Parameters:
        logger (logging.Logger): Logger to use.
        operation (str): Name of the operation being retried.
        attempt (int): Current attempt number.
        max_attempts (int): Maximum number of attempts.
        delay (float): Delay before next retry in seconds.
        error (Exception, optional): The error that caused the retry.

    Example:
        >>> logger = logging.getLogger(__name__)
        >>> log_retry(logger, "API call", attempt=2, max_attempts=3, delay=5.0)
    """
    error_msg = f" (error: {error})" if error else ""
    logger.warning(
        f"{operation} retry {attempt}/{max_attempts}, "
        f"waiting {delay:.1f}s before next attempt{error_msg}"
    )
```

______________________________________________________________________

## Validation

Shared validation functions and constants for agent name validation, chat input validation, tool name sanitization, and tool output sanitization.

## `validate_agent_name(name)`

Validate agent name format.

Agent names must:

- Not be empty
- Be 64 characters or less
- Start with a letter (a-z, A-Z)
- Contain only alphanumeric characters, hyphens, and underscores

Parameters:

| Name   | Type  | Description                | Default    |
| ------ | ----- | -------------------------- | ---------- |
| `name` | `str` | The agent name to validate | *required* |

Returns:

| Type  | Description                                   |
| ----- | --------------------------------------------- |
| `str` | The validated agent name (unchanged if valid) |

Raises:

| Type         | Description              |
| ------------ | ------------------------ |
| `ValueError` | If agent name is invalid |

Source code in `src/holodeck/lib/validation.py`

```
def validate_agent_name(name: str) -> str:
    """Validate agent name format.

    Agent names must:
    - Not be empty
    - Be 64 characters or less
    - Start with a letter (a-z, A-Z)
    - Contain only alphanumeric characters, hyphens, and underscores

    Args:
        name: The agent name to validate

    Returns:
        The validated agent name (unchanged if valid)

    Raises:
        ValueError: If agent name is invalid
    """
    if not name:
        raise ValueError("Agent name cannot be empty")
    if len(name) > AGENT_NAME_MAX_LENGTH:
        raise ValueError(
            f"Agent name must be {AGENT_NAME_MAX_LENGTH} characters or less"
        )
    if not name[0].isalpha():
        raise ValueError("Agent name must start with a letter")
    if not AGENT_NAME_PATTERN.match(name):
        raise ValueError(
            "Agent name must start with a letter and contain only "
            "alphanumeric characters, hyphens, and underscores"
        )
    return name
```

## `ValidationPipeline(max_length=10000)`

Extensible validation pipeline for user input.

Initialize the pipeline with a max length constraint.

Source code in `src/holodeck/lib/validation.py`

```
def __init__(self, max_length: int = 10_000) -> None:
    """Initialize the pipeline with a max length constraint."""
    self.max_length = max_length
```

### `validate(message)`

Validate a message and return (is_valid, error_message).

Source code in `src/holodeck/lib/validation.py`

```
def validate(self, message: str | None) -> tuple[bool, str | None]:
    """Validate a message and return (is_valid, error_message)."""
    if message is None:
        return False, "Message cannot be empty."

    stripped = message.strip()
    if not stripped:
        return False, "Message cannot be empty."

    if len(stripped) > self.max_length:
        return False, "Message exceeds 10,000 characters."

    if CONTROL_CHAR_RE.search(stripped):
        return False, "Message contains control characters."

    try:
        stripped.encode("utf-8")
    except UnicodeEncodeError:
        return False, "Message must be valid UTF-8."

    return True, None
```

## `sanitize_tool_output(output, max_length=5000)`

Remove control/ANSI sequences and truncate long outputs.

Source code in `src/holodeck/lib/validation.py`

```
def sanitize_tool_output(output: str, max_length: int = 5_000) -> str:
    """Remove control/ANSI sequences and truncate long outputs."""
    cleaned = ANSI_ESCAPE_RE.sub("", output)
    cleaned = CONTROL_CHAR_RE.sub("", cleaned)

    if len(cleaned) > max_length:
        cleaned = cleaned[:max_length] + "... (output truncated)"

    return cleaned
```

## `sanitize_tool_name(name)`

Sanitize a string to be a valid tool name.

Tool names must match pattern ^[0-9A-Za-z\_]+$ (alphanumeric and underscores). This function:

1. Replaces all invalid characters with underscores
1. Collapses multiple consecutive underscores into one
1. Strips leading/trailing underscores
1. Validates the result matches the required pattern

Parameters:

| Name   | Type  | Description                  | Default    |
| ------ | ----- | ---------------------------- | ---------- |
| `name` | `str` | Raw name string to sanitize. | *required* |

Returns:

| Type  | Description                                                             |
| ----- | ----------------------------------------------------------------------- |
| `str` | Sanitized name containing only alphanumeric characters and underscores. |

Raises:

| Type         | Description                                                |
| ------------ | ---------------------------------------------------------- |
| `ValueError` | If the input is empty or results in an empty/invalid name. |

Examples:

```
>>> sanitize_tool_name("server-filesystem")
'server_filesystem'
>>> sanitize_tool_name("foo--bar")
'foo_bar'
>>> sanitize_tool_name("my.tool.name")
'my_tool_name'
>>> sanitize_tool_name("  spaces  ")
'spaces'
```

Source code in `src/holodeck/lib/validation.py`

```
def sanitize_tool_name(name: str) -> str:
    """Sanitize a string to be a valid tool name.

    Tool names must match pattern ^[0-9A-Za-z_]+$ (alphanumeric and underscores).
    This function:
    1. Replaces all invalid characters with underscores
    2. Collapses multiple consecutive underscores into one
    3. Strips leading/trailing underscores
    4. Validates the result matches the required pattern

    Args:
        name: Raw name string to sanitize.

    Returns:
        Sanitized name containing only alphanumeric characters and underscores.

    Raises:
        ValueError: If the input is empty or results in an empty/invalid name.

    Examples:
        >>> sanitize_tool_name("server-filesystem")
        'server_filesystem'
        >>> sanitize_tool_name("foo--bar")
        'foo_bar'
        >>> sanitize_tool_name("my.tool.name")
        'my_tool_name'
        >>> sanitize_tool_name("  spaces  ")
        'spaces'
    """
    if not name:
        raise ValueError("Tool name cannot be empty")

    # Replace all invalid characters with underscores
    sanitized = INVALID_TOOL_NAME_CHARS.sub("_", name)

    # Collapse multiple consecutive underscores into one
    sanitized = MULTIPLE_UNDERSCORES.sub("_", sanitized)

    # Strip leading/trailing underscores
    sanitized = sanitized.strip("_")

    # Validate result is not empty
    if not sanitized:
        raise ValueError(
            f"Tool name '{name}' contains no valid characters. "
            "Names must contain at least one alphanumeric character."
        )

    # Final validation against pattern
    if not TOOL_NAME_PATTERN.match(sanitized):
        raise ValueError(
            f"Sanitized tool name '{sanitized}' does not match required pattern. "
            "Names must contain only alphanumeric characters and underscores."
        )

    return sanitized
```

______________________________________________________________________

## Chat History

Utilities for extracting tool information from agent execution results.

## `extract_tool_names(tool_calls)`

Extract tool names from tool calls list.

Tool calls are represented as list of dicts with 'name' and 'arguments' keys.

Parameters:

| Name         | Type                   | Description                         | Default    |
| ------------ | ---------------------- | ----------------------------------- | ---------- |
| `tool_calls` | `list[dict[str, Any]]` | List of tool call dicts from agent. | *required* |

Returns:

| Type        | Description                          |
| ----------- | ------------------------------------ |
| `list[str]` | List of tool names that were called. |

Source code in `src/holodeck/lib/chat_history_utils.py`

```
def extract_tool_names(tool_calls: list[dict[str, Any]]) -> list[str]:
    """Extract tool names from tool calls list.

    Tool calls are represented as list of dicts with 'name' and 'arguments' keys.

    Args:
        tool_calls: List of tool call dicts from agent.

    Returns:
        List of tool names that were called.
    """
    return [call.get("name", "") for call in tool_calls if "name" in call]
```

______________________________________________________________________

## Definition Extraction

Data structures for representing extracted definitions from documents. Definitions are key terms and their explanations used to enhance search results with contextual information.

## `DefinitionEntry(id, source_path, term, term_normalized, definition_text, source_section, exceptions=list())`

An extracted definition from a document.

Represents a term definition extracted from a document, including the term itself, its definition text, and metadata about where it was found.

Attributes:

| Name              | Type        | Description                                           |
| ----------------- | ----------- | ----------------------------------------------------- |
| `id`              | `str`       | Unique identifier for this definition entry           |
| `source_path`     | `str`       | Path to the source document containing the definition |
| `term`            | `str`       | The term being defined (original casing)              |
| `term_normalized` | `str`       | Lowercase normalized term for case-insensitive lookup |
| `definition_text` | `str`       | The full definition text explaining the term          |
| `source_section`  | `str`       | Section ID or heading where the definition was found  |
| `exceptions`      | `list[str]` | List of exceptions or exclusions to the definition    |

Example

> > > entry = DefinitionEntry( ... id="policy_md_def_force_majeure", ... source_path="/docs/policy.md", ... term="Force Majeure", ... term_normalized="force majeure", ... definition_text="Any event beyond the reasonable control...", ... source_section="1.2 Definitions", ... exceptions=["acts of negligence", "breach of contract"], ... )

______________________________________________________________________

## UI Utilities

Terminal detection, color output, and spinner animation utilities for the CLI layer.

### Colors

## `ANSIColors`

ANSI color escape codes for terminal output.

Provides standard ANSI color codes that can be used with the colorize() function or applied directly to strings.

Attributes:

| Name     | Type | Description                                   |
| -------- | ---- | --------------------------------------------- |
| `GREEN`  |      | Bright green color (for success indicators).  |
| `RED`    |      | Bright red color (for failure indicators).    |
| `YELLOW` |      | Bright yellow color (for warnings).           |
| `RESET`  |      | Reset code to restore default terminal color. |

## `colorize(text, color, force_tty=None)`

Apply ANSI color codes to text if in TTY mode.

Wraps text with the specified color code and reset sequence, but only if stdout is connected to a terminal. This ensures clean output in CI/CD logs and file redirects.

Parameters:

| Name        | Type   | Description                                        | Default                                                         |
| ----------- | ------ | -------------------------------------------------- | --------------------------------------------------------------- |
| `text`      | `str`  | Text to colorize.                                  | *required*                                                      |
| `color`     | `str`  | ANSI color code to apply (e.g., ANSIColors.GREEN). | *required*                                                      |
| `force_tty` | \`bool | None\`                                             | Override TTY detection (for testing). None uses auto-detection. |

Returns:

| Type  | Description                                          |
| ----- | ---------------------------------------------------- |
| `str` | Colorized text if in TTY mode, plain text otherwise. |

Source code in `src/holodeck/lib/ui/colors.py`

```
def colorize(text: str, color: str, force_tty: bool | None = None) -> str:
    """Apply ANSI color codes to text if in TTY mode.

    Wraps text with the specified color code and reset sequence,
    but only if stdout is connected to a terminal. This ensures
    clean output in CI/CD logs and file redirects.

    Args:
        text: Text to colorize.
        color: ANSI color code to apply (e.g., ANSIColors.GREEN).
        force_tty: Override TTY detection (for testing). None uses auto-detection.

    Returns:
        Colorized text if in TTY mode, plain text otherwise.
    """
    use_colors = force_tty if force_tty is not None else is_tty()
    if not use_colors:
        return text
    return f"{color}{text}{ANSIColors.RESET}"
```

### Spinner

## `SpinnerMixin`

Mixin providing spinner animation functionality.

Provides braille spinner characters and rotation logic for progress indicators. Classes using this mixin should initialize \_spinner_index = 0 in their **init**.

Class Attributes

SPINNER_CHARS: List of braille characters for spinner animation.

Instance Attributes

\_spinner_index: Current position in spinner rotation (must be initialized).

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

### Terminal Detection

## `is_tty()`

Check if stdout is connected to a terminal.

Used to determine whether to use rich formatting (colors, spinners) or plain text output suitable for CI/CD logs.

Returns:

| Type   | Description                                                      |
| ------ | ---------------------------------------------------------------- |
| `bool` | True if stdout is a TTY (interactive terminal), False otherwise. |

Source code in `src/holodeck/lib/ui/terminal.py`

```
def is_tty() -> bool:
    """Check if stdout is connected to a terminal.

    Used to determine whether to use rich formatting (colors, spinners)
    or plain text output suitable for CI/CD logs.

    Returns:
        True if stdout is a TTY (interactive terminal), False otherwise.
    """
    return sys.stdout.isatty()
```

______________________________________________________________________

## Error Hierarchy

Exception classes for HoloDeck library operations. All exceptions inherit from `HoloDeckError`, enabling generic error handling with `except HoloDeckError`.

## `HoloDeckError`

Bases: `Exception`

Base exception for all HoloDeck library errors.

This is the parent class for all exceptions raised by the holodeck library. Users can catch this to handle any library error generically.

## `ValidationError`

Bases: `HoloDeckError`

Raised when validation fails.

This exception is raised when:

- Input validation fails
- Schema validation fails
- Configuration is invalid

Attributes:

| Name      | Type | Description                           |
| --------- | ---- | ------------------------------------- |
| `message` |      | Description of the validation failure |

## `InitError`

Bases: `HoloDeckError`

Raised when initialization fails.

This exception is raised when:

- Project initialization fails
- Directory creation fails
- File writing fails
- Template rendering fails

Attributes:

| Name      | Type | Description                               |
| --------- | ---- | ----------------------------------------- |
| `message` |      | Description of the initialization failure |

## `TemplateError`

Bases: `HoloDeckError`

Raised when template processing fails.

This exception is raised when:

- Template manifest is malformed or missing
- Jinja2 rendering fails
- Generated content doesn't validate

Attributes:

| Name      | Type | Description                         |
| --------- | ---- | ----------------------------------- |
| `message` |      | Description of the template failure |

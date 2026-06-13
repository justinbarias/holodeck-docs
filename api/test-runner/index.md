# Test Execution Framework API

The test runner orchestrates the complete test execution pipeline for HoloDeck agents, from configuration resolution through agent invocation, evaluation, and result reporting.

The framework follows a sequential flow:

1. Load agent configuration from YAML
1. Resolve execution configuration (CLI > YAML > env > defaults)
1. Initialize components (FileProcessor, AgentBackend, Evaluators)
1. Execute each test case (file processing, agent invocation, tool validation, evaluation)
1. Generate a `TestReport` with summary statistics

______________________________________________________________________

## Executor

The executor module coordinates all stages of test execution. It owns configuration resolution, evaluator creation, agent invocation, and report generation. Agents are driven through the provider-agnostic `AgentBackend` interface — the executor auto-selects the correct backend (Claude or OpenAI Agents) via `BackendSelector` based on `model.provider`, or uses one injected by the caller.

### TestExecutor

## `TestExecutor(agent_config_path, execution_config=None, file_processor=None, evaluators=None, config_loader=None, progress_callback=None, on_test_start=None, force_ingest=False, agent_config=None, resolved_execution_config=None, backend=None)`

Executor for running agent test cases.

Orchestrates the complete test execution flow:

1. Loads agent configuration from YAML file
1. Resolves execution configuration (CLI > YAML > env > defaults)
1. Initializes components (FileProcessor, AgentBackend, Evaluators)
1. Executes test cases sequentially
1. Generates test report with results and summary

Attributes:

| Name                | Type | Description                                       |
| ------------------- | ---- | ------------------------------------------------- |
| `agent_config_path` |      | Path to agent configuration YAML file             |
| `cli_config`        |      | Execution config from CLI flags (optional)        |
| `agent_config`      |      | Loaded agent configuration                        |
| `config`            |      | Resolved execution configuration                  |
| `file_processor`    |      | FileProcessor instance                            |
| `evaluators`        |      | Dictionary of evaluator instances by metric name  |
| `config_loader`     |      | ConfigLoader instance                             |
| `progress_callback` |      | Optional callback function for progress reporting |

Initialize test executor with optional dependency injection.

Follows dependency injection pattern for testability. Dependencies can be:

- Injected explicitly (for testing with mocks)
- Created automatically using factory methods (for normal usage)

When `backend` is provided, the executor uses that `AgentBackend.invoke_once()` path directly. When it is omitted, the executor auto-selects a backend via `BackendSelector` at execution time (normal CLI usage).

Parameters:

| Name                        | Type                             | Description                                      | Default                                                                                                       |
| --------------------------- | -------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------------------------------------------- |
| `agent_config_path`         | `str`                            | Path to agent configuration file                 | *required*                                                                                                    |
| `execution_config`          | \`ExecutionConfig                | None\`                                           | Optional execution config from CLI flags                                                                      |
| `file_processor`            | \`FileProcessor                  | None\`                                           | Optional FileProcessor instance (auto-created if None)                                                        |
| `evaluators`                | \`dict[str, BaseEvaluator]       | None\`                                           | Optional dict of evaluator instances (auto-created if None)                                                   |
| `config_loader`             | \`ConfigLoader                   | None\`                                           | Optional ConfigLoader instance (auto-created if None)                                                         |
| `progress_callback`         | \`Callable\[[TestResult], None\] | None\`                                           | Optional callback function called after each test. Called with TestResult instance. Use for progress display. |
| `force_ingest`              | `bool`                           | Force re-ingestion of vector store source files. | `False`                                                                                                       |
| `agent_config`              | \`Agent                          | None\`                                           | Optional pre-loaded Agent config (auto-loaded if None)                                                        |
| `resolved_execution_config` | \`ExecutionConfig                | None\`                                           | Optional pre-resolved execution config (auto-resolved if None)                                                |
| `backend`                   | \`AgentBackend                   | None\`                                           | Optional AgentBackend instance. When provided, the executor uses it directly instead of auto-selecting one.   |

Source code in `src/holodeck/lib/test_runner/executor.py`

```
def __init__(
    self,
    agent_config_path: str,
    execution_config: ExecutionConfig | None = None,
    file_processor: FileProcessor | None = None,
    evaluators: dict[str, BaseEvaluator] | None = None,
    config_loader: ConfigLoader | None = None,
    progress_callback: Callable[[TestResult], None] | None = None,
    on_test_start: Callable[[TestCaseModel], None] | None = None,
    force_ingest: bool = False,
    agent_config: Agent | None = None,
    resolved_execution_config: ExecutionConfig | None = None,
    backend: AgentBackend | None = None,
) -> None:
    """Initialize test executor with optional dependency injection.

    Follows dependency injection pattern for testability. Dependencies can be:
    - Injected explicitly (for testing with mocks)
    - Created automatically using factory methods (for normal usage)

    When ``backend`` is provided, the executor uses that
    ``AgentBackend.invoke_once()`` path directly. When it is omitted, the
    executor auto-selects a backend via ``BackendSelector`` at execution
    time (normal CLI usage).

    Args:
        agent_config_path: Path to agent configuration file
        execution_config: Optional execution config from CLI flags
        file_processor: Optional FileProcessor instance (auto-created if None)
        evaluators: Optional dict of evaluator instances (auto-created if None)
        config_loader: Optional ConfigLoader instance (auto-created if None)
        progress_callback: Optional callback function called after each test.
                          Called with TestResult instance. Use for progress display.
        force_ingest: Force re-ingestion of vector store source files.
        agent_config: Optional pre-loaded Agent config (auto-loaded if None)
        resolved_execution_config: Optional pre-resolved execution config
                                   (auto-resolved if None)
        backend: Optional AgentBackend instance. When provided, the executor
                 uses it directly instead of auto-selecting one.
    """
    self.agent_config_path = agent_config_path
    self.cli_config = execution_config
    self.config_loader = config_loader or ConfigLoader()
    self.progress_callback = progress_callback
    self.on_test_start = on_test_start
    self._force_ingest = force_ingest
    self._backend: AgentBackend | None = backend

    logger.debug(f"Initializing TestExecutor for config: {agent_config_path}")

    # Use injected agent config or load from file
    self.agent_config = agent_config or self._load_agent_config()

    # Use injected resolved config or resolve from hierarchy
    self.config = resolved_execution_config or self._resolve_execution_config()

    # Use injected dependencies or create defaults
    logger.debug("Initializing FileProcessor component")
    self.file_processor = file_processor or self._create_file_processor()

    # The agent invocation path is the provider-agnostic AgentBackend. When
    # no backend is injected it is auto-selected lazily via BackendSelector
    # in _ensure_backend_initialized() at execution time.
    if self._backend is not None:
        logger.debug("Using injected AgentBackend")
    else:
        logger.debug(
            "No backend injected — will auto-select via BackendSelector "
            "at execution time"
        )

    logger.debug("Initializing Evaluators component")
    self.evaluators = evaluators or self._create_evaluators()

    logger.info(
        f"TestExecutor initialized: {len(self.evaluators)} evaluators, "
        f"timeout={self.config.llm_timeout}s"
    )
```

### `execute_tests()`

Execute all test cases and generate report.

Per-test-case concurrency is controlled by `parallel_test_cases` (feature 032 FR-009a). Turns within a single multi-turn case stay strictly sequential. Progress callbacks and reporter emission are serialised behind an asyncio.Lock so per-test-case output blocks don't interleave mid-record.

Source code in `src/holodeck/lib/test_runner/executor.py`

```
async def execute_tests(self) -> TestReport:
    """Execute all test cases and generate report.

    Per-test-case concurrency is controlled by `parallel_test_cases`
    (feature 032 FR-009a). Turns within a single multi-turn case stay
    strictly sequential. Progress callbacks and reporter emission are
    serialised behind an asyncio.Lock so per-test-case output blocks
    don't interleave mid-record.
    """
    await self._ensure_backend_initialized()

    test_cases = self.agent_config.test_cases or []
    parallel = self.config.parallel_test_cases or 1
    logger.info(
        f"Starting test execution: {len(test_cases)} test cases "
        f"(parallel_test_cases={parallel})"
    )

    # Preserve input order in the output regardless of completion order.
    results: list[TestResult | None] = [None] * len(test_cases)
    semaphore = asyncio.Semaphore(parallel)
    emit_lock = asyncio.Lock()

    async def run_one(idx: int, test_case: TestCaseModel) -> None:
        async with semaphore:
            logger.debug(
                f"Executing test {idx + 1}/{len(test_cases)}: {test_case.name}"
            )
            if self.on_test_start:
                self.on_test_start(test_case)
            result = await self._execute_single_test(test_case)
            results[idx] = result
            async with emit_lock:
                status = "PASS" if result.passed else "FAIL"
                logger.info(
                    f"Test {idx + 1}/{len(test_cases)} {status}: "
                    f"{test_case.name} ({result.execution_time_ms}ms)"
                )
                if self.progress_callback:
                    self.progress_callback(result)

    if parallel <= 1:
        for idx, test_case in enumerate(test_cases):
            await run_one(idx, test_case)
    else:
        await asyncio.gather(*[run_one(i, tc) for i, tc in enumerate(test_cases)])

    # Collect non-None in order (all slots should be filled).
    ordered: list[TestResult] = [r for r in results if r is not None]

    logger.debug("Generating test report")
    return self._generate_report(ordered)
```

### `shutdown()`

Shutdown executor and cleanup resources.

Must be called from the same task context where the executor was used to properly cleanup MCP plugins and other async resources.

Source code in `src/holodeck/lib/test_runner/executor.py`

```
async def shutdown(self) -> None:
    """Shutdown executor and cleanup resources.

    Must be called from the same task context where the executor was used
    to properly cleanup MCP plugins and other async resources.
    """
    try:
        logger.debug("TestExecutor shutting down")
        if self._backend is not None:
            await self._backend.teardown()
        logger.debug("TestExecutor shutdown complete")
    except Exception as e:
        logger.error(f"Error during TestExecutor shutdown: {e}")
```

### validate_tool_calls

Standalone helper that checks actual tool calls against expected tool names using substring matching. Returns `True`, `False`, or `None` (when validation is skipped).

## `validate_tool_calls(actual, expected)`

Validate actual tool calls against expected tools.

Tool call validation checks that each expected tool name is found within at least one actual tool call. This uses substring matching - if any actual tool name contains the expected tool name, it's considered a match.

Parameters:

| Name       | Type        | Description                                 | Default                                                             |
| ---------- | ----------- | ------------------------------------------- | ------------------------------------------------------------------- |
| `actual`   | `list[str]` | List of tool names actually called by agent | *required*                                                          |
| `expected` | \`list[str] | None\`                                      | List of expected tool names from test case (None = skip validation) |

Returns:

| Type   | Description |
| ------ | ----------- |
| \`bool | None\`      |
| \`bool | None\`      |
| \`bool | None\`      |

Examples:

- expected=["search"], actual=["vectorstore-search"] -> True
- expected=["search", "fetch"], actual=["search_tool", "fetch_data"] -> True
- expected=["search"], actual=["fetch"] -> False

Source code in `src/holodeck/lib/test_runner/executor.py`

```
def validate_tool_calls(
    actual: list[str],
    expected: list[str] | None,
) -> bool | None:
    """Validate actual tool calls against expected tools.

    Tool call validation checks that each expected tool name is found within
    at least one actual tool call. This uses substring matching - if any actual
    tool name contains the expected tool name, it's considered a match.

    Args:
        actual: List of tool names actually called by agent
        expected: List of expected tool names from test case (None = skip validation)

    Returns:
        True if all expected tools are found (substring match) in actual
        False if any expected tool is not found in any actual tool
        None if expected is None (validation skipped)

    Examples:
        - expected=["search"], actual=["vectorstore-search"] -> True
        - expected=["search", "fetch"], actual=["search_tool", "fetch_data"] -> True
        - expected=["search"], actual=["fetch"] -> False
    """
    if expected is None:
        return None

    def is_expected_found(expected_tool: str) -> bool:
        """Check if expected tool name is found in any actual tool call."""
        return any(
            tool_name_matches(expected_tool, actual_tool) for actual_tool in actual
        )

    matched = all(is_expected_found(exp) for exp in expected)

    logger.debug(
        f"Tool validation: expected={expected}, actual={actual}, " f"matched={matched}"
    )

    return matched
```

### RAGEvaluatorConstructor

Protocol that defines the common constructor signature shared by all RAG evaluator classes (`FaithfulnessEvaluator`, `ContextualRelevancyEvaluator`, etc.). Used as the value type in `RAG_EVALUATOR_MAP`.

## `RAGEvaluatorConstructor`

Bases: `Protocol`

Protocol for RAG evaluator constructors with full type safety.

Defines the common constructor signature for all RAG evaluators. The actual evaluators may have additional parameters with defaults (timeout, retry_config) but this Protocol captures what we use.

### RAG_EVALUATOR_MAP

Module-level dictionary mapping `RAGMetricType` enum members to their evaluator constructor. Eliminates repetitive `if/elif` chains when creating RAG evaluators.

## `RAG_EVALUATOR_MAP = {RAGMetricType.FAITHFULNESS: FaithfulnessEvaluator, RAGMetricType.CONTEXTUAL_RELEVANCY: ContextualRelevancyEvaluator, RAGMetricType.CONTEXTUAL_PRECISION: ContextualPrecisionEvaluator, RAGMetricType.CONTEXTUAL_RECALL: ContextualRecallEvaluator, RAGMetricType.ANSWER_RELEVANCY: AnswerRelevancyEvaluator}`

______________________________________________________________________

## Agent Invocation

Agent execution is delegated to the backend layer rather than a test-runner-owned agent factory. The executor invokes `AgentBackend.invoke_once()` (or runs a multi-turn session) and normalizes the returned `ExecutionResult` — response text, tool calls, tool results, and token usage — for evaluation. See the [Backend Abstraction](https://docs.useholodeck.ai/api/backends/index.md) reference for `AgentBackend`, `BackendSelector`, and `ExecutionResult`.

______________________________________________________________________

## Reporter

Generates comprehensive Markdown reports from `TestReport` objects, including summary tables, per-test sections, metric details, tool-usage validation, and file metadata.

### generate_markdown_report

## `generate_markdown_report(report)`

Generate a comprehensive markdown report from test results.

Creates a formatted markdown document containing:

- Report header with agent name and metadata
- Summary statistics table
- Detailed test result sections with all fields

Parameters:

| Name     | Type         | Description                                                  | Default    |
| -------- | ------------ | ------------------------------------------------------------ | ---------- |
| `report` | `TestReport` | The TestReport containing all test results and summary data. | *required* |

Returns:

| Type  | Description                                                   |
| ----- | ------------------------------------------------------------- |
| `str` | A formatted markdown string ready for display or file output. |

Source code in `src/holodeck/lib/test_runner/reporter.py`

```
def generate_markdown_report(report: TestReport) -> str:
    """Generate a comprehensive markdown report from test results.

    Creates a formatted markdown document containing:
    - Report header with agent name and metadata
    - Summary statistics table
    - Detailed test result sections with all fields

    Parameters:
        report: The TestReport containing all test results and summary data.

    Returns:
        A formatted markdown string ready for display or file output.
    """
    lines: list[str] = []

    # Header
    lines.append(f"# Test Report: {report.agent_name}\n")
    lines.append(f"**Configuration:** `{report.agent_config_path}`")
    lines.append(f"**Generated:** {report.timestamp}")
    lines.append(f"**HoloDeck Version:** {report.holodeck_version}")

    if report.environment:
        env_parts = []
        if "python_version" in report.environment:
            env_parts.append(report.environment["python_version"])
        if "os" in report.environment:
            env_parts.append(report.environment["os"])
        if env_parts:
            lines.append(f"**Environment:** {' on '.join(env_parts)}")

    lines.append("")

    # Summary section
    lines.append("## Summary\n")
    lines.append(_format_summary_table(report.summary))
    lines.append("")

    # Average metric scores (if available)
    if report.summary.average_scores:
        lines.append("### Average Metric Scores\n")
        score_lines = ["| Metric | Average Score | Scale |"]
        score_lines.append("|--------|----------------|-------|")
        for metric, avg_score in report.summary.average_scores.items():
            score_lines.append(f"| {metric} | {avg_score:.2f} | 0-1 |")
        lines.append("\n".join(score_lines))
        lines.append("")

    lines.append("---\n")

    # Test results
    lines.append("## Test Results\n")
    for result in report.results:
        lines.append(_format_test_section(result))
        lines.append("")

    # Final summary
    lines.append("---\n")
    lines.append("## Report Summary\n")
    status_emoji_pass = "✅" if report.summary.passed > 0 else ""
    status_emoji_fail = "❌" if report.summary.failed > 0 else ""
    lines.append(
        f"{status_emoji_pass} **{report.summary.passed} tests passed** | "
        f"{status_emoji_fail} **{report.summary.failed} tests failed** | "
        f"**Pass Rate: {report.summary.pass_rate:.2f}%**\n"
    )

    return "\n".join(lines)
```

______________________________________________________________________

## Progress

Real-time progress display with TTY detection. Interactive terminals get colored symbols and spinners; CI/CD environments get plain-text output compatible with log aggregation systems.

### ProgressIndicator

## `ProgressIndicator(total_tests, quiet=False, verbose=False)`

Bases: `SpinnerMixin`

Displays progress during test execution with TTY-aware formatting.

Detects whether stdout is a terminal (TTY) and adjusts output accordingly:

- TTY (interactive): Colored symbols, spinners, ANSI formatting
- Non-TTY (CI/CD): Plain text, compatible with log aggregation systems

Inherits spinner animation from SpinnerMixin.

Attributes:

| Name           | Type | Description                                  |
| -------------- | ---- | -------------------------------------------- |
| `total_tests`  |      | Total number of tests to execute             |
| `current_test` |      | Number of tests completed so far             |
| `passed`       |      | Number of tests that passed                  |
| `failed`       |      | Number of tests that failed                  |
| `quiet`        |      | Suppress progress output (only show summary) |
| `verbose`      |      | Show detailed output including timing        |

Initialize progress indicator.

Parameters:

| Name          | Type   | Description                                           | Default    |
| ------------- | ------ | ----------------------------------------------------- | ---------- |
| `total_tests` | `int`  | Total number of tests to execute                      | *required* |
| `quiet`       | `bool` | If True, suppress progress output (only show summary) | `False`    |
| `verbose`     | `bool` | If True, show detailed output with timing information | `False`    |

Source code in `src/holodeck/lib/test_runner/progress.py`

```
def __init__(
    self,
    total_tests: int,
    quiet: bool = False,
    verbose: bool = False,
) -> None:
    """Initialize progress indicator.

    Args:
        total_tests: Total number of tests to execute
        quiet: If True, suppress progress output (only show summary)
        verbose: If True, show detailed output with timing information
    """
    self.total_tests = total_tests
    self.current_test = 0
    self.passed = 0
    self.failed = 0
    self.quiet = quiet
    self.verbose = verbose
    self.test_results: list[TestResult] = []
    self.start_time = datetime.now()
    self._spinner_index = 0
```

### `get_progress_line()`

Get current progress display line.

Returns:

| Type  | Description                                           |
| ----- | ----------------------------------------------------- |
| `str` | Progress string showing current test count and status |
| `str` | Empty string if quiet mode is enabled                 |

Source code in `src/holodeck/lib/test_runner/progress.py`

```
def get_progress_line(self) -> str:
    """Get current progress display line.

    Returns:
        Progress string showing current test count and status
        Empty string if quiet mode is enabled
    """
    if self.quiet and self.current_test < self.total_tests:
        return ""

    if self.current_test == 0:
        return ""

    # Get the last test result
    last_result = self.test_results[-1]

    # Format: "Test X/Y: [symbol] TestName"
    progress = f"Test {self.current_test}/{self.total_tests}"

    if is_tty():
        status = self._format_test_status(last_result)
        return f"{progress}: {status}"
    else:
        # Plain text format for CI/CD
        status = self._format_test_status(last_result)
        return f"[{progress}] {status}"
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

Get current spinner line for running test.

Returns:

| Type  | Description                                              |
| ----- | -------------------------------------------------------- |
| `str` | Formatted spinner string (e.g. "⠋ Test 1/5: Running...") |

Source code in `src/holodeck/lib/test_runner/progress.py`

```
def get_spinner_line(self) -> str:
    """Get current spinner line for running test.

    Returns:
        Formatted spinner string (e.g. "⠋ Test 1/5: Running...")
    """
    if not is_tty() or self.quiet:
        return ""

    spinner = self.get_spinner_char()
    next_test = self.current_test + 1

    # Ensure we don't exceed total tests in display
    if next_test > self.total_tests:
        next_test = self.total_tests

    return f"{spinner} Test {next_test}/{self.total_tests}: Running..."
```

### `get_summary()`

Get summary statistics for all completed tests.

Returns:

| Type  | Description                                             |
| ----- | ------------------------------------------------------- |
| `str` | Formatted summary string with pass/fail counts and rate |

Source code in `src/holodeck/lib/test_runner/progress.py`

```
def get_summary(self) -> str:
    """Get summary statistics for all completed tests.

    Returns:
        Formatted summary string with pass/fail counts and rate
    """
    if self.total_tests == 0:
        return "No tests to execute"

    # Calculate pass rate
    pass_rate = (self.passed / self.total_tests) * 100

    # Format summary
    summary_lines: list[str] = []
    summary_lines.append("")
    summary_lines.append("=" * 60)

    if is_tty():
        # TTY: Use colored symbols
        if self.failed == 0:
            pass_symbol = colorize("\u2713", ANSIColors.GREEN)  # ✓
        else:
            pass_symbol = colorize("\u26a0", ANSIColors.RED)  # ⚠
        summary_lines.append(
            f"{pass_symbol} Test Results: {self.passed}/{self.total_tests} passed "
            f"({pass_rate:.1f}%)"
        )
    else:
        # Plain text
        summary_lines.append(
            f"Test Results: {self.passed}/{self.total_tests} passed "
            f"({pass_rate:.1f}%)"
        )

    if self.failed > 0:
        summary_lines.append(f"  Failed: {self.failed}")

    # Add timing if available
    if hasattr(self, "start_time") and self.start_time:
        elapsed = (datetime.now() - self.start_time).total_seconds()
        summary_lines.append(f"  Duration: {elapsed:.2f}s")

    # Verbose mode: show per-test details with metrics
    if self.verbose and self.test_results:
        summary_lines.append("")
        summary_lines.append("Test Details:")
        for i, result in enumerate(self.test_results, 1):
            if result.passed:
                check = colorize("\u2713", ANSIColors.GREEN)  # ✓
            else:
                check = colorize("\u2717", ANSIColors.RED)  # ✗
            name = result.test_name or f"Test {i}"
            timing = (
                f" ({result.execution_time_ms}ms)"
                if result.execution_time_ms
                else ""
            )
            summary_lines.append(f"  {check} {name}{timing}")

            # Display metric results for each test
            if result.metric_results:
                for metric in result.metric_results:
                    metric_symbol = self._format_metric_symbol(metric.passed)
                    score_str = self._format_metric_score(metric)
                    summary_lines.append(
                        f"      {metric_symbol} {metric.metric_name}: {score_str}"
                    )
                    # Show reasoning if available (DeepEval metrics only)
                    if metric.reasoning:
                        summary_lines.append(
                            f"        Reasoning: {metric.reasoning}"
                        )

    summary_lines.append("=" * 60)

    return "\n".join(summary_lines)
```

### `start_test(test_name)`

Mark a test as started.

Parameters:

| Name        | Type  | Description               | Default    |
| ----------- | ----- | ------------------------- | ---------- |
| `test_name` | `str` | Name of the test starting | *required* |

Source code in `src/holodeck/lib/test_runner/progress.py`

```
def start_test(self, test_name: str) -> None:
    """Mark a test as started.

    Args:
        test_name: Name of the test starting
    """
    self.current_test_name = test_name
```

### `update(result)`

Update progress with a completed test result.

Parameters:

| Name     | Type         | Description                               | Default    |
| -------- | ------------ | ----------------------------------------- | ---------- |
| `result` | `TestResult` | TestResult instance from a completed test | *required* |

Source code in `src/holodeck/lib/test_runner/progress.py`

```
def update(self, result: "TestResult") -> None:
    """Update progress with a completed test result.

    Args:
        result: TestResult instance from a completed test
    """
    self.current_test += 1
    self.test_results.append(result)

    if result.passed:
        self.passed += 1
    else:
        self.failed += 1
```

______________________________________________________________________

## Eval Kwargs Builder

Type-safe construction of evaluation keyword arguments based on each evaluator's `ParamSpec`. Handles the parameter-name divergence between evaluator families (Azure AI/NLP use `response`/`query`; DeepEval uses `actual_output`/`input`).

### EvalKwargsBuilder

## `EvalKwargsBuilder(agent_response, input_query=None, ground_truth=None, file_content=None, retrieval_context=None)`

Builder for evaluation kwargs based on evaluator specifications.

Constructs eval_kwargs dictionaries based on:

1. Evaluator's ParamSpec (required/optional parameters)
1. Available data (test case inputs, file content, tool results)
1. Evaluator type (DeepEval vs Azure AI/NLP param names)

Example

> > > builder = EvalKwargsBuilder( ... input_query="What is X?", ... agent_response="X is...", ... ground_truth="X is the answer", ... file_content="Context from files...", ... retrieval_context=["chunk1", "chunk2"], ... ) kwargs = builder.build_for(evaluator) result = await evaluator.evaluate(\*\*kwargs)

Initialize the kwargs builder.

Parameters:

| Name                | Type        | Description                              | Default                                        |
| ------------------- | ----------- | ---------------------------------------- | ---------------------------------------------- |
| `agent_response`    | `str`       | Agent's response text (always required). | *required*                                     |
| `input_query`       | \`str       | None\`                                   | User's input query.                            |
| `ground_truth`      | \`str       | None\`                                   | Expected ground truth answer.                  |
| `file_content`      | \`str       | None\`                                   | Combined content from processed files.         |
| `retrieval_context` | \`list[str] | None\`                                   | List of retrieved text chunks for RAG metrics. |

Source code in `src/holodeck/lib/test_runner/eval_kwargs_builder.py`

```
def __init__(
    self,
    agent_response: str,
    input_query: str | None = None,
    ground_truth: str | None = None,
    file_content: str | None = None,
    retrieval_context: list[str] | None = None,
) -> None:
    """Initialize the kwargs builder.

    Args:
        agent_response: Agent's response text (always required).
        input_query: User's input query.
        ground_truth: Expected ground truth answer.
        file_content: Combined content from processed files.
        retrieval_context: List of retrieved text chunks for RAG metrics.
    """
    self._agent_response = agent_response
    self._input_query = input_query
    self._ground_truth = ground_truth
    self._file_content = file_content
    self._retrieval_context = retrieval_context
```

### `build_for(evaluator)`

Build eval_kwargs for a specific evaluator.

The method:

1. Gets the evaluator's PARAM_SPEC
1. Determines if it uses DeepEval param names (input/actual_output) or standard names (query/response)
1. Builds kwargs with the appropriate keys

Parameters:

| Name        | Type            | Description                                 | Default    |
| ----------- | --------------- | ------------------------------------------- | ---------- |
| `evaluator` | `BaseEvaluator` | The evaluator instance to build kwargs for. | *required* |

Returns:

| Type             | Description                                          |
| ---------------- | ---------------------------------------------------- |
| `dict[str, Any]` | Dictionary of kwargs ready for evaluator.evaluate(). |

Source code in `src/holodeck/lib/test_runner/eval_kwargs_builder.py`

```
def build_for(self, evaluator: BaseEvaluator) -> dict[str, Any]:
    """Build eval_kwargs for a specific evaluator.

    The method:
    1. Gets the evaluator's PARAM_SPEC
    2. Determines if it uses DeepEval param names (input/actual_output)
       or standard names (query/response)
    3. Builds kwargs with the appropriate keys

    Args:
        evaluator: The evaluator instance to build kwargs for.

    Returns:
        Dictionary of kwargs ready for evaluator.evaluate().
    """
    spec = evaluator.get_param_spec()
    uses_deepeval = spec.uses_deepeval_params()

    kwargs: dict[str, Any] = {}

    # Add response/actual_output (always included)
    if uses_deepeval:
        kwargs["actual_output"] = self._agent_response
    else:
        kwargs["response"] = self._agent_response

    # Add query/input if needed and available
    needs_query = self._should_include(
        EvalParam.QUERY, spec
    ) or self._should_include(EvalParam.INPUT, spec)
    if needs_query and self._input_query:
        if uses_deepeval:
            kwargs["input"] = self._input_query
        else:
            kwargs["query"] = self._input_query

    # Add ground_truth/expected_output if needed and available
    needs_ground_truth = self._should_include(
        EvalParam.GROUND_TRUTH, spec
    ) or self._should_include(EvalParam.EXPECTED_OUTPUT, spec)
    if needs_ground_truth and self._ground_truth:
        if uses_deepeval:
            kwargs["expected_output"] = self._ground_truth
        else:
            kwargs["ground_truth"] = self._ground_truth

    # Add context if evaluator uses it and available
    needs_context = spec.uses_context or self._should_include(
        EvalParam.CONTEXT, spec
    )
    if needs_context and self._file_content:
        kwargs["context"] = self._file_content

    # Add retrieval_context if evaluator uses it and available
    needs_retrieval = spec.uses_retrieval_context or self._should_include(
        EvalParam.RETRIEVAL_CONTEXT, spec
    )
    if needs_retrieval and self._retrieval_context:
        kwargs["retrieval_context"] = self._retrieval_context

    logger.debug(
        f"Built kwargs for {evaluator.name}: "
        f"keys={list(kwargs.keys())}, uses_deepeval={uses_deepeval}"
    )

    return kwargs
```

### build_retrieval_context_from_tools

Extracts retrieval context strings from tool results, filtering to only those tools marked as retrieval tools.

## `build_retrieval_context_from_tools(tool_results, retrieval_tool_names)`

Extract retrieval context from tool results.

Only includes results from tools marked as retrieval tools.

Parameters:

| Name                   | Type                   | Description                                                                                                                   | Default    |
| ---------------------- | ---------------------- | ----------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `tool_results`         | `list[dict[str, Any]]` | List of tool result dicts with 'name' and 'result' keys. The 'result' value can be a string, list of strings, or other types. | *required* |
| `retrieval_tool_names` | `set[str]`             | Set of tool names that provide retrieval context.                                                                             | *required* |

Returns:

| Type        | Description |
| ----------- | ----------- |
| \`list[str] | None\`      |

Source code in `src/holodeck/lib/test_runner/eval_kwargs_builder.py`

```
def build_retrieval_context_from_tools(
    tool_results: list[dict[str, Any]],
    retrieval_tool_names: set[str],
) -> list[str] | None:
    """Extract retrieval context from tool results.

    Only includes results from tools marked as retrieval tools.

    Args:
        tool_results: List of tool result dicts with 'name' and 'result' keys.
            The 'result' value can be a string, list of strings, or other types.
        retrieval_tool_names: Set of tool names that provide retrieval context.

    Returns:
        List of retrieval context strings, or None if none found.
    """
    context: list[str] = []
    for result in tool_results:
        tool_name = result.get("name", "")
        result_content: Any = result.get("result", "")
        if tool_name in retrieval_tool_names and result_content:
            if isinstance(result_content, str):
                context.append(result_content)
            elif isinstance(result_content, list):
                # Safely convert list items to strings, filtering out empty values
                item: Any
                for item in result_content:
                    if item is not None:
                        str_item = str(item)
                        if str_item:
                            context.append(str_item)

    return context if context else None
```

______________________________________________________________________

## Example Usage

```
from holodeck.lib.test_runner.executor import TestExecutor
from holodeck.lib.test_runner.reporter import generate_markdown_report
from holodeck.lib.test_runner.progress import ProgressIndicator
from holodeck.config.loader import ConfigLoader

# Load agent configuration
loader = ConfigLoader()
agent = loader.load_agent_yaml("agent.yaml")

# Set up progress tracking
progress = ProgressIndicator(total_tests=len(agent.test_cases or []))

# Create executor with progress callback
executor = TestExecutor(
    agent_config_path="agent.yaml",
    progress_callback=progress.update,
    on_test_start=lambda tc: progress.start_test(tc.name or "unnamed"),
)

# Run all test cases
report = await executor.execute_tests()

# Display summary
print(progress.get_summary())

# Generate markdown report
markdown = generate_markdown_report(report)
with open("report.md", "w") as f:
    f.write(markdown)
```

### Using EvalKwargsBuilder directly

```
from holodeck.lib.test_runner.eval_kwargs_builder import (
    EvalKwargsBuilder,
    build_retrieval_context_from_tools,
)

# Build retrieval context from tool results
retrieval_ctx = build_retrieval_context_from_tools(
    tool_results=[
        {"name": "search_kb", "result": "Refund policy allows 30-day returns."},
        {"name": "get_user", "result": "User: Alice"},
    ],
    retrieval_tool_names={"search_kb"},
)

# Build kwargs for an evaluator
builder = EvalKwargsBuilder(
    agent_response="We offer 30-day returns.",
    input_query="What is your refund policy?",
    ground_truth="30-day money-back guarantee on all products.",
    retrieval_context=retrieval_ctx,
)
kwargs = builder.build_for(evaluator)
result = await evaluator.evaluate(**kwargs)
```

______________________________________________________________________

## Related Documentation

- [Data Models](https://docs.useholodeck.ai/api/models/index.md) -- Test case and result Pydantic models
- [Evaluation Framework](https://docs.useholodeck.ai/api/evaluators/index.md) -- Metrics, evaluators, and `ParamSpec`
- [Configuration Loading](https://docs.useholodeck.ai/api/config-loader/index.md) -- `ConfigLoader` and resolution hierarchy
- [Backend Abstraction](https://docs.useholodeck.ai/api/backends/index.md) -- `AgentBackend`, `BackendSelector`, and `ExecutionResult`

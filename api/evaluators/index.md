# Evaluation Framework API

HoloDeck provides a flexible evaluation framework for measuring agent response quality. The framework supports three tiers of metrics:

1. **Standard NLP Metrics** -- Traditional text-comparison metrics (BLEU, ROUGE, METEOR) that require no LLM
1. **Azure AI Metrics** -- AI-assisted quality metrics via Azure AI Evaluation SDK (groundedness, relevance, coherence, fluency, similarity)
1. **DeepEval Metrics** -- LLM-as-a-judge evaluation with multi-provider support (G-Eval custom criteria, RAG pipeline metrics)

All evaluators share a common base class with retry logic, timeout handling, and a unified parameter specification system.

______________________________________________________________________

## Architecture Overview

```
BaseEvaluator (base.py)
├── BLEUEvaluator (nlp_metrics.py)
├── ROUGEEvaluator (nlp_metrics.py)
├── METEOREvaluator (nlp_metrics.py)
├── AzureAIEvaluator (azure_ai.py)
│   ├── GroundednessEvaluator
│   ├── RelevanceEvaluator
│   ├── CoherenceEvaluator
│   ├── FluencyEvaluator
│   └── SimilarityEvaluator
└── DeepEvalBaseEvaluator (deepeval/base.py)
    ├── GEvalEvaluator (deepeval/geval.py)
    ├── FaithfulnessEvaluator (deepeval/faithfulness.py)
    ├── AnswerRelevancyEvaluator (deepeval/answer_relevancy.py)
    ├── ContextualRelevancyEvaluator (deepeval/contextual_relevancy.py)
    ├── ContextualPrecisionEvaluator (deepeval/contextual_precision.py)
    └── ContextualRecallEvaluator (deepeval/contextual_recall.py)
```

______________________________________________________________________

## Configuration Models

Evaluation metrics are configured in `agent.yaml` using Pydantic models from `holodeck.models.evaluation`. The `metrics` list uses a discriminated union on the `type` field (`standard`, `geval`, or `rag`).

### YAML Configuration Example

```
evaluations:
  model:                          # Default LLM for all LLM-based metrics
    provider: openai
    name: gpt-4o
    temperature: 0.0
  metrics:
    # Standard NLP metric (no LLM required)
    - type: standard
      metric: bleu
      threshold: 0.4

    # G-Eval custom criteria (LLM-as-judge)
    - type: geval
      name: Helpfulness
      criteria: "Evaluate if the response provides actionable information"
      evaluation_params: [actual_output, input]
      threshold: 0.7

    # RAG pipeline metric
    - type: rag
      metric_type: faithfulness
      threshold: 0.8
      include_reason: true
```

### EvaluationConfig

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

### MetricType

The discriminated union that routes to the correct metric model based on the `type` field:

```
MetricType = Annotated[
    EvaluationMetric | GEvalMetric | RAGMetric,
    Field(discriminator="type"),
]
```

### EvaluationMetric

Standard metric configuration (`type: standard`).

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

### GEvalMetric

G-Eval custom criteria configuration (`type: geval`).

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

### RAGMetric

RAG pipeline metric configuration (`type: rag`).

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

### RAGMetricType

## `RAGMetricType`

Bases: `str`, `Enum`

RAG pipeline evaluation metric types.

These metrics evaluate the quality of Retrieval-Augmented Generation (RAG) pipelines by assessing various aspects of retrieval and response generation.

______________________________________________________________________

## Base Framework

All evaluators inherit from `BaseEvaluator`, which provides retry logic with exponential backoff, timeout handling, and a parameter specification system.

### BaseEvaluator

## `BaseEvaluator(timeout=60.0, retry_config=None)`

Bases: `ABC`

Abstract base class for all evaluation metrics.

This class provides retry logic, timeout handling, and a common interface for all evaluators (AI-assisted and NLP metrics).

Attributes:

| Name           | Type        | Description                                                           |
| -------------- | ----------- | --------------------------------------------------------------------- |
| `timeout`      |             | Timeout in seconds for evaluation (default: 60s, None for no timeout) |
| `retry_config` |             | Configuration for retry logic with exponential backoff                |
| `name`         | `str`       | Evaluator name (defaults to class name)                               |
| `PARAM_SPEC`   | `ParamSpec` | Class attribute declaring required/optional parameters                |

Example

> > > class MyEvaluator(BaseEvaluator): ... PARAM_SPEC = ParamSpec( ... required=frozenset({EvalParam.RESPONSE, EvalParam.QUERY}) ... ) ... async def \_evaluate_impl(self, \*\*kwargs): ... return {"score": 0.85, "passed": True}
> > >
> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate(query="test", response="answer")

Initialize base evaluator.

Parameters:

| Name           | Type          | Description | Default                                             |
| -------------- | ------------- | ----------- | --------------------------------------------------- |
| `timeout`      | \`float       | None\`      | Timeout in seconds (None for no timeout)            |
| `retry_config` | \`RetryConfig | None\`      | Retry configuration (uses defaults if not provided) |

Source code in `src/holodeck/lib/evaluators/base.py`

```
def __init__(
    self,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize base evaluator.

    Args:
        timeout: Timeout in seconds (None for no timeout)
        retry_config: Retry configuration (uses defaults if not provided)
    """
    self.timeout = timeout
    self.retry_config = retry_config or RetryConfig()

    logger.debug(
        f"Evaluator initialized: {self.name}, timeout={timeout}s, "
        f"max_retries={self.retry_config.max_retries}"
    )
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### RetryConfig

## `RetryConfig`

Bases: `BaseModel`

Configuration for retry logic with exponential backoff.

Attributes:

| Name               | Type    | Description                                                  |
| ------------------ | ------- | ------------------------------------------------------------ |
| `max_retries`      | `int`   | Maximum number of retry attempts (default: 3)                |
| `base_delay`       | `float` | Base delay in seconds for exponential backoff (default: 2.0) |
| `max_delay`        | `float` | Maximum delay between retries in seconds (default: 60.0)     |
| `exponential_base` | `float` | Exponential base for backoff calculation (default: 2.0)      |

Example

> > > config = RetryConfig(max_retries=3, base_delay=2.0)
> > >
> > > ### Delays will be: 2.0s, 4.0s, 8.0s

### `validate_delays(v)`

Validate delays are positive.

Source code in `src/holodeck/lib/evaluators/base.py`

```
@field_validator("base_delay", "max_delay")
@classmethod
def validate_delays(cls, v: float) -> float:
    """Validate delays are positive."""
    if v <= 0:
        raise ValueError("Delays must be positive")
    return v
```

### `validate_max_retries(v)`

Validate max_retries is non-negative.

Source code in `src/holodeck/lib/evaluators/base.py`

```
@field_validator("max_retries")
@classmethod
def validate_max_retries(cls, v: int) -> int:
    """Validate max_retries is non-negative."""
    if v < 0:
        raise ValueError("max_retries must be non-negative")
    return v
```

### EvaluationError

## `EvaluationError`

Bases: `Exception`

Exception raised when evaluation fails after all retry attempts.

______________________________________________________________________

## Parameter Specification

The `param_spec` module defines a standard way for evaluators to declare their required and optional inputs. This enables the test runner to validate inputs before calling the evaluator.

### EvalParam

## `EvalParam`

Bases: `str`, `Enum`

Standard evaluation parameter names.

Two naming conventions are supported:

- Azure AI / NLP: RESPONSE, QUERY, GROUND_TRUTH
- DeepEval: ACTUAL_OUTPUT, INPUT, EXPECTED_OUTPUT

Both conventions share CONTEXT and RETRIEVAL_CONTEXT.

### ParamSpec

## `ParamSpec`

Bases: `NamedTuple`

Parameter specification for an evaluator.

Declares which parameters an evaluator requires and optionally accepts, plus flags for special context handling.

Attributes:

| Name                     | Type                   | Description                                          |
| ------------------------ | ---------------------- | ---------------------------------------------------- |
| `required`               | `frozenset[EvalParam]` | Parameters that must be provided for evaluation.     |
| `optional`               | `frozenset[EvalParam]` | Parameters that may be provided but aren't required. |
| `uses_context`           | `bool`                 | Whether file content should be passed as context.    |
| `uses_retrieval_context` | `bool`                 | Whether retrieval context from tools is needed.      |

Example

> > > spec = ParamSpec( ... required=frozenset({EvalParam.RESPONSE, EvalParam.QUERY}), ... optional=frozenset({EvalParam.CONTEXT}), ... uses_context=True, ... )

### `uses_deepeval_params()`

Check if this spec uses DeepEval parameter naming convention.

Returns:

| Type   | Description                                                 |
| ------ | ----------------------------------------------------------- |
| `bool` | True if any required or optional param is a DeepEval param. |

Source code in `src/holodeck/lib/evaluators/param_spec.py`

```
def uses_deepeval_params(self) -> bool:
    """Check if this spec uses DeepEval parameter naming convention.

    Returns:
        True if any required or optional param is a DeepEval param.
    """
    all_params = self.required | self.optional
    return bool(all_params & DEEPEVAL_PARAMS)
```

### DEEPEVAL_PARAMS

A `frozenset` of DeepEval-specific parameter names (`INPUT`, `ACTUAL_OUTPUT`, `EXPECTED_OUTPUT`) used by `ParamSpec.uses_deepeval_params()` to detect the DeepEval naming convention.

```
DEEPEVAL_PARAMS = frozenset(
    {EvalParam.INPUT, EvalParam.ACTUAL_OUTPUT, EvalParam.EXPECTED_OUTPUT}
)
```

Two naming conventions exist side-by-side:

| Convention     | Query   | Response        | Reference         |
| -------------- | ------- | --------------- | ----------------- |
| Azure AI / NLP | `query` | `response`      | `ground_truth`    |
| DeepEval       | `input` | `actual_output` | `expected_output` |

Both conventions share `context` and `retrieval_context`.

______________________________________________________________________

## Standard NLP Metrics

Traditional text-comparison metrics that do not require an LLM. All NLP evaluators require `response` and `ground_truth` parameters.

### BLEUEvaluator

Uses SacreBLEU with exponential smoothing. Scores are normalized from SacreBLEU's 0--100 scale to 0.0--1.0.

## `BLEUEvaluator(threshold=None, timeout=60.0, **kwargs)`

Bases: `BaseEvaluator`

BLEU score evaluator using SacreBLEU with smoothing.

BLEU (Bilingual Evaluation Understudy) measures precision of n-gram matches between prediction and reference text. Uses SacreBLEU with exponential smoothing to handle short sentences and avoid zero scores when there are no 4-gram matches.

Score Range: 0.0-1.0 (normalized from SacreBLEU's 0-100 scale) Higher scores indicate better match to reference text.

Attributes:

| Name           | Type | Description                                |
| -------------- | ---- | ------------------------------------------ |
| `threshold`    |      | Minimum passing score (0.0-1.0)            |
| `timeout`      |      | Timeout in seconds for evaluation          |
| `retry_config` |      | Retry configuration for transient failures |

Example

> > > evaluator = BLEUEvaluator(threshold=0.5) result = await evaluator.evaluate( ... response="The cat sat on the mat", ... ground_truth="The cat is on the mat" ... ) print(result["bleu"]) # 0.0-1.0 print(result["passed"]) # True if >= threshold

Initialize BLEU evaluator.

Parameters:

| Name        | Type    | Description                                  | Default                         |
| ----------- | ------- | -------------------------------------------- | ------------------------------- |
| `threshold` | \`float | None\`                                       | Minimum passing score (0.0-1.0) |
| `timeout`   | \`float | None\`                                       | Timeout in seconds              |
| `**kwargs`  | `Any`   | Additional arguments passed to BaseEvaluator | `{}`                            |

Source code in `src/holodeck/lib/evaluators/nlp_metrics.py`

```
def __init__(
    self,
    threshold: float | None = None,
    timeout: float | None = 60.0,
    **kwargs: Any,
) -> None:
    """Initialize BLEU evaluator.

    Args:
        threshold: Minimum passing score (0.0-1.0)
        timeout: Timeout in seconds
        **kwargs: Additional arguments passed to BaseEvaluator
    """
    super().__init__(timeout=timeout, **kwargs)
    self.threshold = threshold
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### ROUGEEvaluator

Returns all three ROUGE variants (`rouge1`, `rouge2`, `rougeL`). The `variant` parameter controls which variant is used for the threshold check.

## `ROUGEEvaluator(threshold=None, variant='rougeL', timeout=60.0, **kwargs)`

Bases: `BaseEvaluator`

ROUGE score evaluator.

ROUGE (Recall-Oriented Understudy for Gisting Evaluation) measures recall of n-gram overlaps between prediction and reference. Commonly used for summarization evaluation.

Variants:

- ROUGE-1: Unigram overlap
- ROUGE-2: Bigram overlap
- ROUGE-L: Longest common subsequence

Score Range: 0.0-1.0 (F1 score) Higher scores indicate better recall of reference text.

Attributes:

| Name           | Type | Description                                                             |
| -------------- | ---- | ----------------------------------------------------------------------- |
| `threshold`    |      | Minimum passing score (0.0-1.0)                                         |
| `variant`      |      | ROUGE variant to use for threshold check ("rouge1", "rouge2", "rougeL") |
| `timeout`      |      | Timeout in seconds for evaluation                                       |
| `retry_config` |      | Retry configuration for transient failures                              |

Example

> > > evaluator = ROUGEEvaluator(threshold=0.6, variant="rougeL") result = await evaluator.evaluate( ... response="The cat sat on the mat", ... ground_truth="The cat is on the mat" ... ) print(result["rouge1"]) # 0.0-1.0 print(result["rouge2"]) # 0.0-1.0 print(result["rougeL"]) # 0.0-1.0 print(result["passed"]) # True if rougeL >= threshold

Initialize ROUGE evaluator.

Parameters:

| Name        | Type    | Description                                                      | Default                         |
| ----------- | ------- | ---------------------------------------------------------------- | ------------------------------- |
| `threshold` | \`float | None\`                                                           | Minimum passing score (0.0-1.0) |
| `variant`   | `str`   | ROUGE variant for threshold check ("rouge1", "rouge2", "rougeL") | `'rougeL'`                      |
| `timeout`   | \`float | None\`                                                           | Timeout in seconds              |
| `**kwargs`  | `Any`   | Additional arguments passed to BaseEvaluator                     | `{}`                            |

Raises:

| Type         | Description             |
| ------------ | ----------------------- |
| `ValueError` | If variant is not valid |

Source code in `src/holodeck/lib/evaluators/nlp_metrics.py`

```
def __init__(
    self,
    threshold: float | None = None,
    variant: str = "rougeL",
    timeout: float | None = 60.0,
    **kwargs: Any,
) -> None:
    """Initialize ROUGE evaluator.

    Args:
        threshold: Minimum passing score (0.0-1.0)
        variant: ROUGE variant for threshold check
            ("rouge1", "rouge2", "rougeL")
        timeout: Timeout in seconds
        **kwargs: Additional arguments passed to BaseEvaluator

    Raises:
        ValueError: If variant is not valid
    """
    super().__init__(timeout=timeout, **kwargs)
    self.threshold = threshold

    valid_variants = {"rouge1", "rouge2", "rougeL"}
    if variant not in valid_variants:
        raise ValueError(f"variant must be one of {valid_variants}, got: {variant}")
    self.variant = variant
    self._metric = None  # Lazy loaded
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### METEOREvaluator

Synonym-aware matching with stemming for better correlation with human judgment.

## `METEOREvaluator(threshold=None, timeout=60.0, **kwargs)`

Bases: `BaseEvaluator`

METEOR score evaluator.

METEOR (Metric for Evaluation of Translation with Explicit ORdering) measures translation quality using synonym matching, stemming, and paraphrase detection. Provides better correlation with human judgment than BLEU.

Score Range: 0.0-1.0 Higher scores indicate better semantic match to reference text.

Attributes:

| Name           | Type | Description                                |
| -------------- | ---- | ------------------------------------------ |
| `threshold`    |      | Minimum passing score (0.0-1.0)            |
| `timeout`      |      | Timeout in seconds for evaluation          |
| `retry_config` |      | Retry configuration for transient failures |

Example

> > > evaluator = METEOREvaluator(threshold=0.7) result = await evaluator.evaluate( ... response="The automobile is red", ... ground_truth="The car is red" ... ) print(result["meteor"]) # Higher than BLEU due to synonym handling print(result["passed"]) # True if >= threshold

Initialize METEOR evaluator.

Parameters:

| Name        | Type    | Description                                  | Default                         |
| ----------- | ------- | -------------------------------------------- | ------------------------------- |
| `threshold` | \`float | None\`                                       | Minimum passing score (0.0-1.0) |
| `timeout`   | \`float | None\`                                       | Timeout in seconds              |
| `**kwargs`  | `Any`   | Additional arguments passed to BaseEvaluator | `{}`                            |

Source code in `src/holodeck/lib/evaluators/nlp_metrics.py`

```
def __init__(
    self,
    threshold: float | None = None,
    timeout: float | None = 60.0,
    **kwargs: Any,
) -> None:
    """Initialize METEOR evaluator.

    Args:
        threshold: Minimum passing score (0.0-1.0)
        timeout: Timeout in seconds
        **kwargs: Additional arguments passed to BaseEvaluator
    """
    super().__init__(timeout=timeout, **kwargs)
    self.threshold = threshold
    self._metric = None  # Lazy loaded
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### NLPMetricsError

## `NLPMetricsError`

Bases: `EvaluationError`

Exception raised when NLP metric computation fails.

### NLP Metrics Usage

```
from holodeck.lib.evaluators.nlp_metrics import BLEUEvaluator, ROUGEEvaluator

bleu = BLEUEvaluator(threshold=0.5)
result = await bleu.evaluate(
    response="The cat sat on the mat",
    ground_truth="The cat is on the mat",
)
print(result["bleu"])    # 0.0-1.0
print(result["passed"])  # True if >= 0.5

rouge = ROUGEEvaluator(threshold=0.6, variant="rougeL")
result = await rouge.evaluate(
    response="The cat sat on the mat",
    ground_truth="The cat is on the mat",
)
print(result["rouge1"], result["rouge2"], result["rougeL"])
```

### NLP Metrics Summary

| Metric            | Score Key                    | Score Range | Use Case                               |
| ----------------- | ---------------------------- | ----------- | -------------------------------------- |
| `BLEUEvaluator`   | `bleu`                       | 0.0--1.0    | Precision-focused n-gram matching      |
| `ROUGEEvaluator`  | `rouge1`, `rouge2`, `rougeL` | 0.0--1.0    | Recall-focused overlap (summarization) |
| `METEOREvaluator` | `meteor`                     | 0.0--1.0    | Synonym-aware semantic similarity      |

______________________________________________________________________

## Azure AI Metrics

AI-assisted quality metrics powered by the Azure AI Evaluation SDK. All Azure evaluators normalize scores from a 1--5 scale to 0.0--1.0.

### ModelConfig

## `ModelConfig`

Bases: `BaseModel`

Azure OpenAI model configuration for evaluators.

Attributes:

| Name               | Type  | Description                                              |
| ------------------ | ----- | -------------------------------------------------------- |
| `azure_endpoint`   | `str` | Azure OpenAI endpoint URL                                |
| `api_key`          | `str` | Azure OpenAI API key                                     |
| `azure_deployment` | `str` | Azure deployment name (e.g., "gpt-4o", "gpt-4o-mini")    |
| `api_version`      | `str` | Azure OpenAI API version (default: "2024-02-15-preview") |

Example

> > > config = ModelConfig( ... azure_endpoint="https://my-resource.openai.azure.com/", ... api_key="my-api-key", ... azure_deployment="gpt-4o" ... )

### AzureAIEvaluator

## `AzureAIEvaluator(model_config, timeout=60.0, retry_config=None)`

Bases: `BaseEvaluator`

Base class for Azure AI Evaluation SDK evaluators.

Provides common functionality for all Azure AI evaluators:

- Model configuration
- Retry logic with exponential backoff
- Timeout handling
- Score normalization (5-point scale to 0-1)

Attributes:

| Name           | Type | Description                                  |
| -------------- | ---- | -------------------------------------------- |
| `model_config` |      | Azure OpenAI model configuration             |
| `timeout`      |      | Timeout in seconds (default: 60s)            |
| `retry_config` |      | Retry configuration with exponential backoff |

Example

> > > config = ModelConfig( ... azure_endpoint="https://test.openai.azure.com/", ... api_key="key", ... azure_deployment="gpt-4o-mini" ... ) evaluator = RelevanceEvaluator(model_config=config) result = await evaluator.evaluate(query="test", response="answer")

Initialize Azure AI evaluator.

Parameters:

| Name           | Type          | Description                      | Default                                                |
| -------------- | ------------- | -------------------------------- | ------------------------------------------------------ |
| `model_config` | `ModelConfig` | Azure OpenAI model configuration | *required*                                             |
| `timeout`      | \`float       | None\`                           | Timeout in seconds (default: 60s, None for no timeout) |
| `retry_config` | \`RetryConfig | None\`                           | Retry configuration (uses defaults if not provided)    |

Source code in `src/holodeck/lib/evaluators/azure_ai.py`

```
def __init__(
    self,
    model_config: ModelConfig,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize Azure AI evaluator.

    Args:
        model_config: Azure OpenAI model configuration
        timeout: Timeout in seconds (default: 60s, None for no timeout)
        retry_config: Retry configuration (uses defaults if not provided)
    """
    super().__init__(timeout=timeout, retry_config=retry_config)
    self.model_config = model_config
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### GroundednessEvaluator

Assesses whether all claims in the response are supported by the provided context. Use an expensive model (e.g., `gpt-4o`) for this critical metric.

## `GroundednessEvaluator(model_config, timeout=60.0, retry_config=None)`

Bases: `AzureAIEvaluator`

Groundedness evaluator using Azure AI Evaluation SDK.

Assesses correspondence between claims in AI-generated answers and source context. Measures factual accuracy by verifying that all claims in the response are supported by the provided context.

Query parameter is optional but recommended for better accuracy.

Scale: 1-5 (normalized to 0.0-1.0)

Example

> > > config = ModelConfig( ... azure_endpoint="https://test.openai.azure.com/", ... api_key="key", ... azure_deployment="gpt-4o" # Use expensive model for critical metric ... ) evaluator = GroundednessEvaluator(model_config=config) result = await evaluator.evaluate( ... query="What is the capital?", ... response="The capital is Paris.", ... context="France's capital is Paris." ... ) print(result["score"]) # 0.0-1.0 0.95

Initialize Azure AI evaluator.

Parameters:

| Name           | Type          | Description                      | Default                                                |
| -------------- | ------------- | -------------------------------- | ------------------------------------------------------ |
| `model_config` | `ModelConfig` | Azure OpenAI model configuration | *required*                                             |
| `timeout`      | \`float       | None\`                           | Timeout in seconds (default: 60s, None for no timeout) |
| `retry_config` | \`RetryConfig | None\`                           | Retry configuration (uses defaults if not provided)    |

Source code in `src/holodeck/lib/evaluators/azure_ai.py`

```
def __init__(
    self,
    model_config: ModelConfig,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize Azure AI evaluator.

    Args:
        model_config: Azure OpenAI model configuration
        timeout: Timeout in seconds (default: 60s, None for no timeout)
        retry_config: Retry configuration (uses defaults if not provided)
    """
    super().__init__(timeout=timeout, retry_config=retry_config)
    self.model_config = model_config
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### RelevanceEvaluator

Measures whether the response directly addresses the user's question.

## `RelevanceEvaluator(model_config, timeout=60.0, retry_config=None)`

Bases: `AzureAIEvaluator`

Relevance evaluator using Azure AI Evaluation SDK.

Measures relevance of response to query. Assesses whether the response directly addresses the user's question or request.

Scale: 1-5 (normalized to 0.0-1.0)

Example

> > > config = ModelConfig( ... azure_endpoint="https://test.openai.azure.com/", ... api_key="key", ... azure_deployment="gpt-4o" # Critical metric ... ) evaluator = RelevanceEvaluator(model_config=config) result = await evaluator.evaluate( ... query="What is ML?", ... response="ML is machine learning, a subset of AI." ... )

Initialize Azure AI evaluator.

Parameters:

| Name           | Type          | Description                      | Default                                                |
| -------------- | ------------- | -------------------------------- | ------------------------------------------------------ |
| `model_config` | `ModelConfig` | Azure OpenAI model configuration | *required*                                             |
| `timeout`      | \`float       | None\`                           | Timeout in seconds (default: 60s, None for no timeout) |
| `retry_config` | \`RetryConfig | None\`                           | Retry configuration (uses defaults if not provided)    |

Source code in `src/holodeck/lib/evaluators/azure_ai.py`

```
def __init__(
    self,
    model_config: ModelConfig,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize Azure AI evaluator.

    Args:
        model_config: Azure OpenAI model configuration
        timeout: Timeout in seconds (default: 60s, None for no timeout)
        retry_config: Retry configuration (uses defaults if not provided)
    """
    super().__init__(timeout=timeout, retry_config=retry_config)
    self.model_config = model_config
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### CoherenceEvaluator

Evaluates logical flow and readability of the response.

## `CoherenceEvaluator(model_config, timeout=60.0, retry_config=None)`

Bases: `AzureAIEvaluator`

Coherence evaluator using Azure AI Evaluation SDK.

Evaluates logical flow and readability. Measures how well the response is organized and whether ideas connect logically.

Scale: 1-5 (normalized to 0.0-1.0)

Example

> > > config = ModelConfig( ... azure_endpoint="https://test.openai.azure.com/", ... api_key="key", ... azure_deployment="gpt-4o-mini" # Less critical metric ... ) evaluator = CoherenceEvaluator(model_config=config) result = await evaluator.evaluate( ... query="Explain X", ... response="X is... Furthermore... In conclusion..." ... )

Initialize Azure AI evaluator.

Parameters:

| Name           | Type          | Description                      | Default                                                |
| -------------- | ------------- | -------------------------------- | ------------------------------------------------------ |
| `model_config` | `ModelConfig` | Azure OpenAI model configuration | *required*                                             |
| `timeout`      | \`float       | None\`                           | Timeout in seconds (default: 60s, None for no timeout) |
| `retry_config` | \`RetryConfig | None\`                           | Retry configuration (uses defaults if not provided)    |

Source code in `src/holodeck/lib/evaluators/azure_ai.py`

```
def __init__(
    self,
    model_config: ModelConfig,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize Azure AI evaluator.

    Args:
        model_config: Azure OpenAI model configuration
        timeout: Timeout in seconds (default: 60s, None for no timeout)
        retry_config: Retry configuration (uses defaults if not provided)
    """
    super().__init__(timeout=timeout, retry_config=retry_config)
    self.model_config = model_config
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### FluencyEvaluator

Assesses grammar, spelling, punctuation, word choice, and sentence structure.

## `FluencyEvaluator(model_config, timeout=60.0, retry_config=None)`

Bases: `AzureAIEvaluator`

Fluency evaluator using Azure AI Evaluation SDK.

Assesses language quality. Measures grammar, spelling, punctuation, word choice, and sentence structure.

Scale: 1-5 (normalized to 0.0-1.0)

Example

> > > config = ModelConfig( ... azure_endpoint="https://test.openai.azure.com/", ... api_key="key", ... azure_deployment="gpt-4o-mini" # Less critical metric ... ) evaluator = FluencyEvaluator(model_config=config) result = await evaluator.evaluate( ... query="Test", ... response="This is a well-written response." ... )

Initialize Azure AI evaluator.

Parameters:

| Name           | Type          | Description                      | Default                                                |
| -------------- | ------------- | -------------------------------- | ------------------------------------------------------ |
| `model_config` | `ModelConfig` | Azure OpenAI model configuration | *required*                                             |
| `timeout`      | \`float       | None\`                           | Timeout in seconds (default: 60s, None for no timeout) |
| `retry_config` | \`RetryConfig | None\`                           | Retry configuration (uses defaults if not provided)    |

Source code in `src/holodeck/lib/evaluators/azure_ai.py`

```
def __init__(
    self,
    model_config: ModelConfig,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize Azure AI evaluator.

    Args:
        model_config: Azure OpenAI model configuration
        timeout: Timeout in seconds (default: 60s, None for no timeout)
        retry_config: Retry configuration (uses defaults if not provided)
    """
    super().__init__(timeout=timeout, retry_config=retry_config)
    self.model_config = model_config
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### SimilarityEvaluator

Compares semantic similarity between response and ground truth.

## `SimilarityEvaluator(model_config, timeout=60.0, retry_config=None)`

Bases: `AzureAIEvaluator`

Similarity evaluator using Azure AI Evaluation SDK.

Compares semantic similarity between response and ground truth. Measures how closely the response matches the expected answer.

Requires ground_truth parameter.

Scale: 1-5 (normalized to 0.0-1.0)

Example

> > > config = ModelConfig( ... azure_endpoint="https://test.openai.azure.com/", ... api_key="key", ... azure_deployment="gpt-4o-mini" ... ) evaluator = SimilarityEvaluator(model_config=config) result = await evaluator.evaluate( ... query="What is 2+2?", ... response="The answer is 4.", ... ground_truth="2+2 equals 4." ... )

Initialize Azure AI evaluator.

Parameters:

| Name           | Type          | Description                      | Default                                                |
| -------------- | ------------- | -------------------------------- | ------------------------------------------------------ |
| `model_config` | `ModelConfig` | Azure OpenAI model configuration | *required*                                             |
| `timeout`      | \`float       | None\`                           | Timeout in seconds (default: 60s, None for no timeout) |
| `retry_config` | \`RetryConfig | None\`                           | Retry configuration (uses defaults if not provided)    |

Source code in `src/holodeck/lib/evaluators/azure_ai.py`

```
def __init__(
    self,
    model_config: ModelConfig,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
) -> None:
    """Initialize Azure AI evaluator.

    Args:
        model_config: Azure OpenAI model configuration
        timeout: Timeout in seconds (default: 60s, None for no timeout)
        retry_config: Retry configuration (uses defaults if not provided)
    """
    super().__init__(timeout=timeout, retry_config=retry_config)
    self.model_config = model_config
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### Azure AI Usage

```
from holodeck.lib.evaluators.azure_ai import (
    ModelConfig,
    GroundednessEvaluator,
    RelevanceEvaluator,
)

config = ModelConfig(
    azure_endpoint="https://my-resource.openai.azure.com/",
    api_key="my-api-key",
    azure_deployment="gpt-4o",
)

groundedness = GroundednessEvaluator(model_config=config)
result = await groundedness.evaluate(
    query="What is the capital of France?",
    response="The capital of France is Paris.",
    context="France is a country in Europe. Its capital is Paris.",
)
print(result["score"])          # 0.0-1.0 (normalized from 1-5)
print(result["groundedness"])   # Raw 1-5 score
print(result["reasoning"])      # LLM explanation
```

### Azure AI Metrics Summary

| Evaluator               | Required Params                     | Optional Params | Score Key      |
| ----------------------- | ----------------------------------- | --------------- | -------------- |
| `GroundednessEvaluator` | `response`, `context`               | `query`         | `groundedness` |
| `RelevanceEvaluator`    | `response`, `query`                 | `context`       | `relevance`    |
| `CoherenceEvaluator`    | `response`, `query`                 | --              | `coherence`    |
| `FluencyEvaluator`      | `response`, `query`                 | --              | `fluency`      |
| `SimilarityEvaluator`   | `response`, `query`, `ground_truth` | --              | `similarity`   |

______________________________________________________________________

## DeepEval Metrics

LLM-as-a-judge evaluation with multi-provider support (OpenAI, Azure OpenAI, Anthropic, Ollama). DeepEval metrics use a different parameter naming convention (`input`, `actual_output`, `expected_output`) but HoloDeck's `DeepEvalBaseEvaluator` also accepts Azure/NLP aliases (`query`, `response`, `ground_truth`).

### DeepEvalModelConfig

## `DeepEvalModelConfig`

Bases: `BaseModel`

Configuration adapter for DeepEval model classes.

This class bridges HoloDeck's LLMProvider configuration to DeepEval's native model classes (GPTModel, AzureOpenAIModel, AnthropicModel, OllamaModel).

The default configuration uses Ollama with gpt-oss:20b for local evaluation without requiring API keys.

Attributes:

| Name              | Type           | Description                                                  |
| ----------------- | -------------- | ------------------------------------------------------------ |
| `provider`        | `ProviderEnum` | LLM provider to use (defaults to Ollama)                     |
| `model_name`      | `str`          | Name of the model (defaults to gpt-oss:20b)                  |
| `api_key`         | \`str          | None\`                                                       |
| `endpoint`        | \`str          | None\`                                                       |
| `api_version`     | \`str          | None\`                                                       |
| `deployment_name` | \`str          | None\`                                                       |
| `temperature`     | `float`        | Temperature for generation (defaults to 0.0 for determinism) |

API Key Behavior

- **OpenAI**: API key can be provided via `api_key` field or the `OPENAI_API_KEY` environment variable. If neither is set, DeepEval's GPTModel will raise an error at runtime.
- **Anthropic**: API key can be provided via `api_key` field or the `ANTHROPIC_API_KEY` environment variable. If neither is set, DeepEval's AnthropicModel will raise an error at runtime.
- **Azure OpenAI**: The `api_key` field is **required** and validated at configuration time (no environment variable fallback).
- **Ollama**: No API key required (local inference).

Example

> > > config = DeepEvalModelConfig() # Default Ollama model = config.to_deepeval_model()
> > >
> > > openai_config = DeepEvalModelConfig( ... provider=ProviderEnum.OPENAI, ... model_name="gpt-4o", ... api_key="sk-..." # Or set OPENAI_API_KEY env var ... )

### `to_deepeval_model()`

Convert configuration to native DeepEval model class.

Returns the appropriate DeepEval model class instance based on the configured provider.

Returns:

| Type            | Description                                          |
| --------------- | ---------------------------------------------------- |
| `DeepEvalModel` | DeepEval model instance (GPTModel, AzureOpenAIModel, |
| `DeepEvalModel` | AnthropicModel, or OllamaModel)                      |

Raises:

| Type         | Description                  |
| ------------ | ---------------------------- |
| `ValueError` | If provider is not supported |

Source code in `src/holodeck/lib/evaluators/deepeval/config.py`

```
def to_deepeval_model(self) -> DeepEvalModel:
    """Convert configuration to native DeepEval model class.

    Returns the appropriate DeepEval model class instance based on
    the configured provider.

    Returns:
        DeepEval model instance (GPTModel, AzureOpenAIModel,
        AnthropicModel, or OllamaModel)

    Raises:
        ValueError: If provider is not supported
    """
    if self.provider == ProviderEnum.OPENAI:
        from deepeval.models import GPTModel

        kwargs: dict[str, Any] = {
            "model": self.model_name,
            "temperature": self.temperature,
        }
        if self.api_key:
            kwargs["api_key"] = self.api_key
        return GPTModel(**kwargs)

    elif self.provider == ProviderEnum.AZURE_OPENAI:
        from deepeval.models import AzureOpenAIModel

        # DeepEval 3.7.x renamed the constructor kwargs:
        #   model_name → model (hard-required; not aliased)
        #   azure_endpoint → base_url (aliased, deprecation warning)
        #   openai_api_version → api_version (not aliased — must rename)
        #   azure_openai_api_key → api_key (aliased, deprecation warning)
        # We use the new names directly to avoid the warnings.
        return AzureOpenAIModel(
            model=self.model_name,
            deployment_name=self.deployment_name,
            base_url=self.endpoint,
            api_version=self.api_version,
            api_key=self.api_key,
            temperature=1.0,  # reasoning models require temperature=1.0
        )

    elif self.provider == ProviderEnum.ANTHROPIC:
        from deepeval.models import AnthropicModel

        kwargs = {
            "model": self.model_name,
            "temperature": self.temperature,
        }
        if self.api_key:
            kwargs["api_key"] = self.api_key
        return AnthropicModel(**kwargs)

    elif self.provider == ProviderEnum.OLLAMA:
        from deepeval.models import OllamaModel

        return OllamaModel(
            model=self.model_name,
            base_url=self.endpoint or "http://localhost:11434",
            temperature=self.temperature,
        )

    else:
        raise ValueError(f"Unsupported provider: {self.provider}")
```

### `validate_provider_requirements()`

Validate that required fields are present for each provider.

Raises:

| Type         | Description                                     |
| ------------ | ----------------------------------------------- |
| `ValueError` | If required fields are missing for the provider |

Source code in `src/holodeck/lib/evaluators/deepeval/config.py`

```
@model_validator(mode="after")
def validate_provider_requirements(self) -> "DeepEvalModelConfig":
    """Validate that required fields are present for each provider.

    Raises:
        ValueError: If required fields are missing for the provider
    """
    if self.provider == ProviderEnum.AZURE_OPENAI:
        if not self.endpoint:
            raise ValueError("endpoint is required for Azure OpenAI provider")
        if not self.deployment_name:
            raise ValueError(
                "deployment_name is required for Azure OpenAI provider"
            )
        if not self.api_key:
            raise ValueError("api_key is required for Azure OpenAI provider")
    return self
```

### DeepEvalBaseEvaluator

## `DeepEvalBaseEvaluator(model_config=None, threshold=0.5, timeout=60.0, retry_config=None, observability_config=None)`

Bases: `BaseEvaluator`

Abstract base class for DeepEval-based evaluators.

This class extends BaseEvaluator to provide DeepEval-specific functionality:

- Model configuration and initialization
- LLMTestCase construction from evaluation inputs
- Result normalization and logging

Subclasses must implement \_create_metric() to return the specific DeepEval metric instance.

Note: DeepEval uses different parameter names than Azure AI/NLP:

- input (not query)
- actual_output (not response)
- expected_output (not ground_truth)

Attributes:

| Name           | Type | Description                                 |
| -------------- | ---- | ------------------------------------------- |
| `model_config` |      | Configuration for the evaluation LLM        |
| `threshold`    |      | Score threshold for pass/fail determination |
| `model`        |      | The initialized DeepEval model instance     |

Example

> > > class MyMetricEvaluator(DeepEvalBaseEvaluator): ... def \_create_metric(self): ... return SomeDeepEvalMetric( ... threshold=self.\_threshold, ... model=self.\_model ... )
> > >
> > > evaluator = MyMetricEvaluator(threshold=0.7) result = await evaluator.evaluate( ... input="What is Python?", ... actual_output="Python is a programming language." ... )

Initialize DeepEval base evaluator.

Parameters:

| Name                   | Type                  | Description                                           | Default                                                                        |
| ---------------------- | --------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------ |
| `model_config`         | \`DeepEvalModelConfig | None\`                                                | Configuration for the evaluation model. Defaults to Ollama with gpt-oss:20b.   |
| `threshold`            | `float`               | Score threshold for pass/fail (0.0-1.0, default: 0.5) | `0.5`                                                                          |
| `timeout`              | \`float               | None\`                                                | Evaluation timeout in seconds (default: 60.0)                                  |
| `retry_config`         | \`RetryConfig         | None\`                                                | Retry configuration for transient failures                                     |
| `observability_config` | \`TracingConfig       | None\`                                                | Tracing configuration for span instrumentation. If None, no spans are created. |

Source code in `src/holodeck/lib/evaluators/deepeval/base.py`

```
def __init__(
    self,
    model_config: DeepEvalModelConfig | None = None,
    threshold: float = 0.5,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
    observability_config: TracingConfig | None = None,
) -> None:
    """Initialize DeepEval base evaluator.

    Args:
        model_config: Configuration for the evaluation model.
                     Defaults to Ollama with gpt-oss:20b.
        threshold: Score threshold for pass/fail (0.0-1.0, default: 0.5)
        timeout: Evaluation timeout in seconds (default: 60.0)
        retry_config: Retry configuration for transient failures
        observability_config: Tracing configuration for span instrumentation.
                             If None, no spans are created.
    """
    super().__init__(timeout=timeout, retry_config=retry_config)
    self._model_config = model_config or DeepEvalModelConfig()
    self._model = self._model_config.to_deepeval_model()
    self._threshold = threshold
    self._observability_config = observability_config

    logger.debug(
        f"DeepEval evaluator initialized: {self.name}, "
        f"provider={self._model_config.provider.value}, "
        f"model={self._model_config.model_name}, "
        f"threshold={threshold}"
    )
```

### `name`

Return evaluator name (class name by default).

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

### DeepEvalError

## `DeepEvalError(message, metric_name, original_error=None, test_case_summary=None)`

Bases: `EvaluationError`

Wraps errors from the DeepEval library with additional context.

This exception provides debugging information when DeepEval metrics fail, including the metric name and a summary of the test case that triggered the error.

Attributes:

| Name                | Type | Description                               |
| ------------------- | ---- | ----------------------------------------- |
| `metric_name`       |      | Name of the DeepEval metric that failed   |
| `original_error`    |      | The underlying exception from DeepEval    |
| `test_case_summary` |      | Truncated input/output data for debugging |

Initialize DeepEvalError with context.

Parameters:

| Name                | Type             | Description                    | Default                                    |
| ------------------- | ---------------- | ------------------------------ | ------------------------------------------ |
| `message`           | `str`            | Human-readable error message   | *required*                                 |
| `metric_name`       | `str`            | Name of the metric that failed | *required*                                 |
| `original_error`    | \`Exception      | None\`                         | The underlying exception from DeepEval     |
| `test_case_summary` | \`dict[str, Any] | None\`                         | Dictionary with truncated test case fields |

Source code in `src/holodeck/lib/evaluators/deepeval/errors.py`

```
def __init__(
    self,
    message: str,
    metric_name: str,
    original_error: Exception | None = None,
    test_case_summary: dict[str, Any] | None = None,
) -> None:
    """Initialize DeepEvalError with context.

    Args:
        message: Human-readable error message
        metric_name: Name of the metric that failed
        original_error: The underlying exception from DeepEval
        test_case_summary: Dictionary with truncated test case fields
    """
    super().__init__(message)
    self.metric_name = metric_name
    self.original_error = original_error
    self.test_case_summary = test_case_summary or {}
```

### ProviderNotSupportedError

## `ProviderNotSupportedError(message, evaluator_type, configured_provider, supported_providers)`

Bases: `EvaluationError`

Raised when an evaluator is used with an incompatible LLM provider.

This error is raised early during evaluator initialization to prevent confusing runtime errors when users misconfigure provider settings.

Attributes:

| Name                  | Type | Description                                            |
| --------------------- | ---- | ------------------------------------------------------ |
| `evaluator_type`      |      | The type of evaluator that requires specific providers |
| `configured_provider` |      | The provider that was incorrectly configured           |
| `supported_providers` |      | List of providers that are supported                   |

Initialize ProviderNotSupportedError with context.

Parameters:

| Name                  | Type        | Description                               | Default    |
| --------------------- | ----------- | ----------------------------------------- | ---------- |
| `message`             | `str`       | Human-readable error message              | *required* |
| `evaluator_type`      | `str`       | The evaluator class that raised the error | *required* |
| `configured_provider` | `str`       | The provider that was configured          | *required* |
| `supported_providers` | `list[str]` | List of valid provider names              | *required* |

Source code in `src/holodeck/lib/evaluators/deepeval/errors.py`

```
def __init__(
    self,
    message: str,
    evaluator_type: str,
    configured_provider: str,
    supported_providers: list[str],
) -> None:
    """Initialize ProviderNotSupportedError with context.

    Args:
        message: Human-readable error message
        evaluator_type: The evaluator class that raised the error
        configured_provider: The provider that was configured
        supported_providers: List of valid provider names
    """
    super().__init__(message)
    self.evaluator_type = evaluator_type
    self.configured_provider = configured_provider
    self.supported_providers = supported_providers
```

______________________________________________________________________

### G-Eval: Custom Criteria

#### GEvalEvaluator

## `GEvalEvaluator(name, criteria, evaluation_params=None, evaluation_steps=None, model_config=None, threshold=0.5, strict_mode=False, timeout=60.0, retry_config=None, observability_config=None)`

Bases: `DeepEvalBaseEvaluator`

G-Eval custom criteria evaluator.

Evaluates LLM outputs against user-defined criteria using the G-Eval algorithm, which combines chain-of-thought prompting with token probability scoring.

G-Eval works in two phases:

1. Step Generation: Auto-generates evaluation steps from the criteria
1. Scoring: Uses the steps to score the test case on a 1-5 scale (normalized to 0-1)

Attributes:

| Name                 | Type | Description                                |
| -------------------- | ---- | ------------------------------------------ |
| `_metric_name`       |      | Custom name for this evaluation metric     |
| `_criteria`          |      | Natural language criteria for evaluation   |
| `_evaluation_params` |      | Test case fields to include in evaluation  |
| `_evaluation_steps`  |      | Optional explicit evaluation steps         |
| `_strict_mode`       |      | Whether to use binary scoring (1.0 or 0.0) |

Example

> > > evaluator = GEvalEvaluator( ... name="Professionalism", ... criteria="Evaluate if the response uses professional language", ... threshold=0.7 ... ) result = await evaluator.evaluate( ... input="Write me an email", ... actual_output="Dear Sir/Madam, ..." ... ) print(result["score"]) # 0.85 print(result["passed"]) # True

Initialize G-Eval evaluator.

Parameters:

| Name                   | Type                  | Description                                              | Default                                                                                                                                                            |
| ---------------------- | --------------------- | -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `name`                 | `str`                 | Metric identifier (e.g., "Correctness", "Helpfulness")   | *required*                                                                                                                                                         |
| `criteria`             | `str`                 | Natural language evaluation criteria                     | *required*                                                                                                                                                         |
| `evaluation_params`    | \`list[str]           | None\`                                                   | Test case fields to include in evaluation. Valid options: ["input", "actual_output", "expected_output", "context", "retrieval_context"] Default: ["actual_output"] |
| `evaluation_steps`     | \`list[str]           | None\`                                                   | Explicit evaluation steps. If None, G-Eval auto-generates steps from the criteria.                                                                                 |
| `model_config`         | \`DeepEvalModelConfig | None\`                                                   | LLM judge configuration. Defaults to Ollama gpt-oss:20b.                                                                                                           |
| `threshold`            | `float`               | Pass/fail score threshold (0.0-1.0). Default: 0.5.       | `0.5`                                                                                                                                                              |
| `strict_mode`          | `bool`                | If True, scores are binary (1.0 or 0.0). Default: False. | `False`                                                                                                                                                            |
| `timeout`              | \`float               | None\`                                                   | Evaluation timeout in seconds. Default: 60.0.                                                                                                                      |
| `retry_config`         | \`RetryConfig         | None\`                                                   | Retry configuration for transient failures.                                                                                                                        |
| `observability_config` | \`TracingConfig       | None\`                                                   | Tracing configuration for span instrumentation. If None, no spans are created.                                                                                     |

Raises:

| Type         | Description                                |
| ------------ | ------------------------------------------ |
| `ValueError` | If invalid evaluation_params are provided. |

Source code in `src/holodeck/lib/evaluators/deepeval/geval.py`

```
def __init__(
    self,
    name: str,
    criteria: str,
    evaluation_params: list[str] | None = None,
    evaluation_steps: list[str] | None = None,
    model_config: DeepEvalModelConfig | None = None,
    threshold: float = 0.5,
    strict_mode: bool = False,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
    observability_config: TracingConfig | None = None,
) -> None:
    """Initialize G-Eval evaluator.

    Args:
        name: Metric identifier (e.g., "Correctness", "Helpfulness")
        criteria: Natural language evaluation criteria
        evaluation_params: Test case fields to include in evaluation.
            Valid options: ["input", "actual_output", "expected_output",
                          "context", "retrieval_context"]
            Default: ["actual_output"]
        evaluation_steps: Explicit evaluation steps. If None, G-Eval
            auto-generates steps from the criteria.
        model_config: LLM judge configuration. Defaults to Ollama gpt-oss:20b.
        threshold: Pass/fail score threshold (0.0-1.0). Default: 0.5.
        strict_mode: If True, scores are binary (1.0 or 0.0). Default: False.
        timeout: Evaluation timeout in seconds. Default: 60.0.
        retry_config: Retry configuration for transient failures.
        observability_config: Tracing configuration for span instrumentation.
                             If None, no spans are created.

    Raises:
        ValueError: If invalid evaluation_params are provided.
    """
    # Validate and set evaluation params before calling super().__init__
    if evaluation_params is None:
        evaluation_params = ["actual_output"]

    # Validate evaluation params
    for param in evaluation_params:
        if param not in VALID_EVALUATION_PARAMS:
            raise ValueError(
                f"Invalid evaluation_param: '{param}'. "
                f"Valid options: {sorted(VALID_EVALUATION_PARAMS)}"
            )

    self._metric_name = name
    self._criteria = criteria
    self._evaluation_params = evaluation_params
    self._evaluation_steps = evaluation_steps
    self._strict_mode = strict_mode

    super().__init__(
        model_config=model_config,
        threshold=threshold,
        timeout=timeout,
        retry_config=retry_config,
        observability_config=observability_config,
    )

    logger.debug(
        f"GEvalEvaluator initialized: name={name}, "
        f"criteria_len={len(criteria)}, "
        f"evaluation_params={evaluation_params}, "
        f"strict_mode={strict_mode}"
    )
```

### `name`

Return the custom metric name.

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

#### G-Eval Usage

```
from holodeck.lib.evaluators.deepeval import GEvalEvaluator, DeepEvalModelConfig
from holodeck.models.llm import ProviderEnum

config = DeepEvalModelConfig(
    provider=ProviderEnum.OPENAI,
    model_name="gpt-4o",
    api_key="sk-...",
)

evaluator = GEvalEvaluator(
    name="Professionalism",
    criteria="Evaluate if the response uses professional language and avoids slang.",
    evaluation_params=["actual_output", "input"],
    evaluation_steps=[
        "Check if the language is formal and professional",
        "Verify no slang or casual expressions are used",
    ],
    model_config=config,
    threshold=0.7,
    strict_mode=False,
)

result = await evaluator.evaluate(
    input="Write a business email",
    actual_output="Dear Sir/Madam, I am writing to inquire about...",
)
print(result["score"])      # 0.0-1.0
print(result["passed"])     # True if >= 0.7
print(result["reasoning"])  # LLM-generated explanation
```

#### G-Eval YAML Configuration

```
evaluations:
  model:
    provider: openai
    name: gpt-4o
    temperature: 0.0
  metrics:
    - type: geval
      name: Professionalism
      criteria: |
        Evaluate if the response uses professional language,
        avoids slang, and maintains a respectful tone.
      evaluation_steps:
        - "Check if the language is formal and professional"
        - "Verify no slang or casual expressions are used"
        - "Assess the overall respectful tone"
      evaluation_params:
        - actual_output
        - input
      threshold: 0.7
      strict_mode: false
```

Valid `evaluation_params` values: `input`, `actual_output`, `expected_output`, `context`, `retrieval_context`.

______________________________________________________________________

### RAG Pipeline Metrics

RAG evaluators measure retrieval-augmented generation quality. All RAG evaluators (except `AnswerRelevancyEvaluator`) require `retrieval_context`.

#### FaithfulnessEvaluator

Detects hallucinations by checking whether the response is supported by the retrieval context.

## `FaithfulnessEvaluator(model_config=None, threshold=0.5, include_reason=True, timeout=60.0, retry_config=None, observability_config=None)`

Bases: `DeepEvalBaseEvaluator`

Faithfulness evaluator for detecting hallucinations.

Detects hallucinations by comparing agent response to retrieval context. Returns a low score if the response contains information not found in the retrieval context (hallucination detected).

Required inputs

- input: User query
- actual_output: Agent response
- retrieval_context: List of retrieved text chunks

Example

> > > evaluator = FaithfulnessEvaluator(threshold=0.8) result = await evaluator.evaluate( ... input="What are the store hours?", ... actual_output="Store is open 24/7.", ... retrieval_context=["Store hours: Mon-Fri 9am-5pm"] ... ) print(result["score"]) # Low score (hallucination detected)

Attributes:

| Name              | Type | Description                              |
| ----------------- | ---- | ---------------------------------------- |
| `_include_reason` |      | Whether to include reasoning in results. |

Initialize Faithfulness evaluator.

Parameters:

| Name                   | Type                  | Description                                             | Default                                                                        |
| ---------------------- | --------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `model_config`         | \`DeepEvalModelConfig | None\`                                                  | LLM judge configuration. Defaults to Ollama gpt-oss:20b.                       |
| `threshold`            | `float`               | Pass/fail score threshold (0.0-1.0). Default: 0.5.      | `0.5`                                                                          |
| `include_reason`       | `bool`                | Whether to include reasoning in results. Default: True. | `True`                                                                         |
| `timeout`              | \`float               | None\`                                                  | Evaluation timeout in seconds. Default: 60.0.                                  |
| `retry_config`         | \`RetryConfig         | None\`                                                  | Retry configuration for transient failures.                                    |
| `observability_config` | \`TracingConfig       | None\`                                                  | Tracing configuration for span instrumentation. If None, no spans are created. |

Source code in `src/holodeck/lib/evaluators/deepeval/faithfulness.py`

```
def __init__(
    self,
    model_config: DeepEvalModelConfig | None = None,
    threshold: float = 0.5,
    include_reason: bool = True,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
    observability_config: TracingConfig | None = None,
) -> None:
    """Initialize Faithfulness evaluator.

    Args:
        model_config: LLM judge configuration. Defaults to Ollama gpt-oss:20b.
        threshold: Pass/fail score threshold (0.0-1.0). Default: 0.5.
        include_reason: Whether to include reasoning in results. Default: True.
        timeout: Evaluation timeout in seconds. Default: 60.0.
        retry_config: Retry configuration for transient failures.
        observability_config: Tracing configuration for span instrumentation.
                             If None, no spans are created.
    """
    self._include_reason = include_reason

    super().__init__(
        model_config=model_config,
        threshold=threshold,
        timeout=timeout,
        retry_config=retry_config,
        observability_config=observability_config,
    )

    logger.debug(
        f"FaithfulnessEvaluator initialized: "
        f"provider={self._model_config.provider.value}, "
        f"model={self._model_config.model_name}, "
        f"threshold={threshold}, include_reason={include_reason}"
    )
```

### `name`

Return the metric name.

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

#### AnswerRelevancyEvaluator

Measures whether response statements are relevant to the input query. Does **not** require `retrieval_context`.

## `AnswerRelevancyEvaluator(model_config=None, threshold=0.5, include_reason=True, strict_mode=False, timeout=60.0, retry_config=None, observability_config=None)`

Bases: `DeepEvalBaseEvaluator`

Answer Relevancy evaluator - measures statement relevance to input.

Evaluates how relevant the response statements are to the input query. Unlike other RAG metrics, this does NOT require retrieval_context.

Required inputs

- input: User query
- actual_output: Agent response

Example

> > > evaluator = AnswerRelevancyEvaluator(threshold=0.7) result = await evaluator.evaluate( ... input="What is the return policy?", ... actual_output="We offer a 30-day full refund at no extra cost." ... ) print(result["score"]) # High score if relevant

Attributes:

| Name              | Type | Description                                 |
| ----------------- | ---- | ------------------------------------------- |
| `_include_reason` |      | Whether to include reasoning in results.    |
| `_strict_mode`    |      | Whether to use binary scoring (1.0 or 0.0). |

Initialize Answer Relevancy evaluator.

Parameters:

| Name                   | Type                  | Description                                             | Default                                                                        |
| ---------------------- | --------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `model_config`         | \`DeepEvalModelConfig | None\`                                                  | LLM judge configuration. Defaults to Ollama gpt-oss:20b.                       |
| `threshold`            | `float`               | Pass/fail score threshold (0.0-1.0). Default: 0.5.      | `0.5`                                                                          |
| `include_reason`       | `bool`                | Whether to include reasoning in results. Default: True. | `True`                                                                         |
| `strict_mode`          | `bool`                | Binary scoring mode (1.0 or 0.0 only). Default: False.  | `False`                                                                        |
| `timeout`              | \`float               | None\`                                                  | Evaluation timeout in seconds. Default: 60.0.                                  |
| `retry_config`         | \`RetryConfig         | None\`                                                  | Retry configuration for transient failures.                                    |
| `observability_config` | \`TracingConfig       | None\`                                                  | Tracing configuration for span instrumentation. If None, no spans are created. |

Source code in `src/holodeck/lib/evaluators/deepeval/answer_relevancy.py`

```
def __init__(
    self,
    model_config: DeepEvalModelConfig | None = None,
    threshold: float = 0.5,
    include_reason: bool = True,
    strict_mode: bool = False,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
    observability_config: TracingConfig | None = None,
) -> None:
    """Initialize Answer Relevancy evaluator.

    Args:
        model_config: LLM judge configuration. Defaults to Ollama gpt-oss:20b.
        threshold: Pass/fail score threshold (0.0-1.0). Default: 0.5.
        include_reason: Whether to include reasoning in results. Default: True.
        strict_mode: Binary scoring mode (1.0 or 0.0 only). Default: False.
        timeout: Evaluation timeout in seconds. Default: 60.0.
        retry_config: Retry configuration for transient failures.
        observability_config: Tracing configuration for span instrumentation.
                             If None, no spans are created.
    """
    self._include_reason = include_reason
    self._strict_mode = strict_mode

    super().__init__(
        model_config=model_config,
        threshold=threshold,
        timeout=timeout,
        retry_config=retry_config,
        observability_config=observability_config,
    )

    logger.debug(
        f"AnswerRelevancyEvaluator initialized: "
        f"provider={self._model_config.provider.value}, "
        f"model={self._model_config.model_name}, "
        f"threshold={threshold}, include_reason={include_reason}, "
        f"strict_mode={strict_mode}"
    )
```

### `name`

Return the metric name.

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

#### ContextualRelevancyEvaluator

Measures the proportion of retrieved chunks that are relevant to the query.

## `ContextualRelevancyEvaluator(model_config=None, threshold=0.5, include_reason=True, timeout=60.0, retry_config=None, observability_config=None)`

Bases: `DeepEvalBaseEvaluator`

Contextual Relevancy evaluator for RAG pipelines.

Measures the relevance of retrieved context to the user query. Returns the proportion of chunks that are relevant to the query.

Required inputs

- input: User query
- actual_output: Agent response
- retrieval_context: List of retrieved text chunks

Example

> > > evaluator = ContextualRelevancyEvaluator(threshold=0.6) result = await evaluator.evaluate( ... input="What is the pricing?", ... actual_output="Basic plan is $10/month.", ... retrieval_context=[ ... "Pricing: Basic $10, Pro $25", # Relevant ... "Company founded in 2020", # Irrelevant ... ] ... ) print(result["score"]) # 0.5 (1 of 2 chunks relevant)

Attributes:

| Name              | Type | Description                              |
| ----------------- | ---- | ---------------------------------------- |
| `_include_reason` |      | Whether to include reasoning in results. |

Initialize Contextual Relevancy evaluator.

Parameters:

| Name                   | Type                  | Description                                             | Default                                                                        |
| ---------------------- | --------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `model_config`         | \`DeepEvalModelConfig | None\`                                                  | LLM judge configuration. Defaults to Ollama gpt-oss:20b.                       |
| `threshold`            | `float`               | Pass/fail score threshold (0.0-1.0). Default: 0.5.      | `0.5`                                                                          |
| `include_reason`       | `bool`                | Whether to include reasoning in results. Default: True. | `True`                                                                         |
| `timeout`              | \`float               | None\`                                                  | Evaluation timeout in seconds. Default: 60.0.                                  |
| `retry_config`         | \`RetryConfig         | None\`                                                  | Retry configuration for transient failures.                                    |
| `observability_config` | \`TracingConfig       | None\`                                                  | Tracing configuration for span instrumentation. If None, no spans are created. |

Source code in `src/holodeck/lib/evaluators/deepeval/contextual_relevancy.py`

```
def __init__(
    self,
    model_config: DeepEvalModelConfig | None = None,
    threshold: float = 0.5,
    include_reason: bool = True,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
    observability_config: TracingConfig | None = None,
) -> None:
    """Initialize Contextual Relevancy evaluator.

    Args:
        model_config: LLM judge configuration. Defaults to Ollama gpt-oss:20b.
        threshold: Pass/fail score threshold (0.0-1.0). Default: 0.5.
        include_reason: Whether to include reasoning in results. Default: True.
        timeout: Evaluation timeout in seconds. Default: 60.0.
        retry_config: Retry configuration for transient failures.
        observability_config: Tracing configuration for span instrumentation.
                             If None, no spans are created.
    """
    self._include_reason = include_reason

    super().__init__(
        model_config=model_config,
        threshold=threshold,
        timeout=timeout,
        retry_config=retry_config,
        observability_config=observability_config,
    )

    logger.debug(
        f"ContextualRelevancyEvaluator initialized: "
        f"provider={self._model_config.provider.value}, "
        f"model={self._model_config.model_name}, "
        f"threshold={threshold}, include_reason={include_reason}"
    )
```

### `name`

Return the metric name.

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

#### ContextualPrecisionEvaluator

Evaluates ranking quality -- whether relevant chunks appear before irrelevant ones.

## `ContextualPrecisionEvaluator(model_config=None, threshold=0.5, include_reason=True, timeout=60.0, retry_config=None, observability_config=None)`

Bases: `DeepEvalBaseEvaluator`

Contextual Precision evaluator for RAG pipelines.

Evaluates the ranking quality of retrieved chunks. Measures whether relevant chunks appear before irrelevant ones.

Required inputs

- input: User query
- actual_output: Agent response
- expected_output: Ground truth answer
- retrieval_context: List of retrieved text chunks (order matters)

Example

> > > evaluator = ContextualPrecisionEvaluator(threshold=0.7) result = await evaluator.evaluate( ... input="What is X?", ... actual_output="X is...", ... expected_output="X is the correct definition.", ... retrieval_context=[ ... "Irrelevant info", # Bad: irrelevant first ... "X is the definition", # Good: relevant ... ] ... ) print(result["score"]) # Lower due to poor ranking

Attributes:

| Name              | Type | Description                              |
| ----------------- | ---- | ---------------------------------------- |
| `_include_reason` |      | Whether to include reasoning in results. |

Initialize Contextual Precision evaluator.

Parameters:

| Name                   | Type                  | Description                                             | Default                                                                        |
| ---------------------- | --------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `model_config`         | \`DeepEvalModelConfig | None\`                                                  | LLM judge configuration. Defaults to Ollama gpt-oss:20b.                       |
| `threshold`            | `float`               | Pass/fail score threshold (0.0-1.0). Default: 0.5.      | `0.5`                                                                          |
| `include_reason`       | `bool`                | Whether to include reasoning in results. Default: True. | `True`                                                                         |
| `timeout`              | \`float               | None\`                                                  | Evaluation timeout in seconds. Default: 60.0.                                  |
| `retry_config`         | \`RetryConfig         | None\`                                                  | Retry configuration for transient failures.                                    |
| `observability_config` | \`TracingConfig       | None\`                                                  | Tracing configuration for span instrumentation. If None, no spans are created. |

Source code in `src/holodeck/lib/evaluators/deepeval/contextual_precision.py`

```
def __init__(
    self,
    model_config: DeepEvalModelConfig | None = None,
    threshold: float = 0.5,
    include_reason: bool = True,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
    observability_config: TracingConfig | None = None,
) -> None:
    """Initialize Contextual Precision evaluator.

    Args:
        model_config: LLM judge configuration. Defaults to Ollama gpt-oss:20b.
        threshold: Pass/fail score threshold (0.0-1.0). Default: 0.5.
        include_reason: Whether to include reasoning in results. Default: True.
        timeout: Evaluation timeout in seconds. Default: 60.0.
        retry_config: Retry configuration for transient failures.
        observability_config: Tracing configuration for span instrumentation.
                             If None, no spans are created.
    """
    self._include_reason = include_reason

    super().__init__(
        model_config=model_config,
        threshold=threshold,
        timeout=timeout,
        retry_config=retry_config,
        observability_config=observability_config,
    )

    logger.debug(
        f"ContextualPrecisionEvaluator initialized: "
        f"provider={self._model_config.provider.value}, "
        f"model={self._model_config.model_name}, "
        f"threshold={threshold}, include_reason={include_reason}"
    )
```

### `name`

Return the metric name.

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

#### ContextualRecallEvaluator

Measures retrieval completeness -- whether the context contains all facts needed to produce the expected output.

## `ContextualRecallEvaluator(model_config=None, threshold=0.5, include_reason=True, timeout=60.0, retry_config=None, observability_config=None)`

Bases: `DeepEvalBaseEvaluator`

Contextual Recall evaluator for RAG pipelines.

Measures retrieval completeness against expected output. Evaluates whether retrieval context contains all facts needed to produce the expected output.

Required inputs

- input: User query
- actual_output: Agent response
- expected_output: Ground truth answer
- retrieval_context: List of retrieved text chunks

Example

> > > evaluator = ContextualRecallEvaluator(threshold=0.8) result = await evaluator.evaluate( ... input="List all features", ... actual_output="Features are A and B", ... expected_output="Features are A, B, and C", ... retrieval_context=["Feature A: ...", "Feature B: ..."] ... ) print(result["score"]) # ~0.67 (missing Feature C)

Attributes:

| Name              | Type | Description                              |
| ----------------- | ---- | ---------------------------------------- |
| `_include_reason` |      | Whether to include reasoning in results. |

Initialize Contextual Recall evaluator.

Parameters:

| Name                   | Type                  | Description                                             | Default                                                                        |
| ---------------------- | --------------------- | ------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `model_config`         | \`DeepEvalModelConfig | None\`                                                  | LLM judge configuration. Defaults to Ollama gpt-oss:20b.                       |
| `threshold`            | `float`               | Pass/fail score threshold (0.0-1.0). Default: 0.5.      | `0.5`                                                                          |
| `include_reason`       | `bool`                | Whether to include reasoning in results. Default: True. | `True`                                                                         |
| `timeout`              | \`float               | None\`                                                  | Evaluation timeout in seconds. Default: 60.0.                                  |
| `retry_config`         | \`RetryConfig         | None\`                                                  | Retry configuration for transient failures.                                    |
| `observability_config` | \`TracingConfig       | None\`                                                  | Tracing configuration for span instrumentation. If None, no spans are created. |

Source code in `src/holodeck/lib/evaluators/deepeval/contextual_recall.py`

```
def __init__(
    self,
    model_config: DeepEvalModelConfig | None = None,
    threshold: float = 0.5,
    include_reason: bool = True,
    timeout: float | None = 60.0,
    retry_config: RetryConfig | None = None,
    observability_config: TracingConfig | None = None,
) -> None:
    """Initialize Contextual Recall evaluator.

    Args:
        model_config: LLM judge configuration. Defaults to Ollama gpt-oss:20b.
        threshold: Pass/fail score threshold (0.0-1.0). Default: 0.5.
        include_reason: Whether to include reasoning in results. Default: True.
        timeout: Evaluation timeout in seconds. Default: 60.0.
        retry_config: Retry configuration for transient failures.
        observability_config: Tracing configuration for span instrumentation.
                             If None, no spans are created.
    """
    self._include_reason = include_reason

    super().__init__(
        model_config=model_config,
        threshold=threshold,
        timeout=timeout,
        retry_config=retry_config,
        observability_config=observability_config,
    )

    logger.debug(
        f"ContextualRecallEvaluator initialized: "
        f"provider={self._model_config.provider.value}, "
        f"model={self._model_config.model_name}, "
        f"threshold={threshold}, include_reason={include_reason}"
    )
```

### `name`

Return the metric name.

### `evaluate(**kwargs)`

Evaluate with timeout and retry logic.

This is the main public interface for evaluation. It wraps the implementation with timeout and retry handling.

Parameters:

| Name       | Type  | Description                                                          | Default |
| ---------- | ----- | -------------------------------------------------------------------- | ------- |
| `**kwargs` | `Any` | Evaluation parameters (query, response, context, ground_truth, etc.) | `{}`    |

Returns:

| Type             | Description                  |
| ---------------- | ---------------------------- |
| `dict[str, Any]` | Evaluation result dictionary |

Raises:

| Type              | Description                       |
| ----------------- | --------------------------------- |
| `TimeoutError`    | If evaluation exceeds timeout     |
| `EvaluationError` | If evaluation fails after retries |

Example

> > > evaluator = MyEvaluator(timeout=30.0) result = await evaluator.evaluate( ... query="What is the capital of France?", ... response="The capital of France is Paris.", ... context="France is a country in Europe.", ... ground_truth="Paris" ... ) print(result["score"]) 0.95

Source code in `src/holodeck/lib/evaluators/base.py`

```
async def evaluate(self, **kwargs: Any) -> dict[str, Any]:
    """Evaluate with timeout and retry logic.

    This is the main public interface for evaluation. It wraps the
    implementation with timeout and retry handling.

    Args:
        **kwargs: Evaluation parameters
            (query, response, context, ground_truth, etc.)

    Returns:
        Evaluation result dictionary

    Raises:
        asyncio.TimeoutError: If evaluation exceeds timeout
        EvaluationError: If evaluation fails after retries

    Example:
        >>> evaluator = MyEvaluator(timeout=30.0)
        >>> result = await evaluator.evaluate(
        ...     query="What is the capital of France?",
        ...     response="The capital of France is Paris.",
        ...     context="France is a country in Europe.",
        ...     ground_truth="Paris"
        ... )
        >>> print(result["score"])
        0.95
    """
    logger.debug(f"Starting evaluation: {self.name} (timeout={self.timeout}s)")

    if self.timeout is None:
        # No timeout - evaluate directly with retry
        logger.debug(f"Evaluation {self.name}: no timeout")
        return await self._evaluate_with_retry(**kwargs)

    # Apply timeout using asyncio.wait_for
    try:
        logger.debug(f"Evaluation {self.name}: applying timeout of {self.timeout}s")
        return await asyncio.wait_for(
            self._evaluate_with_retry(**kwargs), timeout=self.timeout
        )
    except TimeoutError:
        logger.error(f"Evaluation {self.name} exceeded timeout of {self.timeout}s")
        raise  # Re-raise timeout error as-is
```

### `get_param_spec()`

Get the parameter specification for this evaluator.

Returns:

| Type        | Description                                                         |
| ----------- | ------------------------------------------------------------------- |
| `ParamSpec` | ParamSpec declaring required/optional parameters and context flags. |

Source code in `src/holodeck/lib/evaluators/base.py`

```
@classmethod
def get_param_spec(cls) -> ParamSpec:
    """Get the parameter specification for this evaluator.

    Returns:
        ParamSpec declaring required/optional parameters and context flags.
    """
    return cls.PARAM_SPEC
```

#### RAG Metrics Usage

```
from holodeck.lib.evaluators.deepeval import (
    FaithfulnessEvaluator,
    AnswerRelevancyEvaluator,
    ContextualRelevancyEvaluator,
    ContextualPrecisionEvaluator,
    ContextualRecallEvaluator,
    DeepEvalModelConfig,
)

config = DeepEvalModelConfig()  # Default: Ollama with gpt-oss:20b

# Faithfulness (hallucination detection)
faithfulness = FaithfulnessEvaluator(model_config=config, threshold=0.8)
result = await faithfulness.evaluate(
    input="What are the store hours?",
    actual_output="Store is open 24/7.",
    retrieval_context=["Store hours: Mon-Fri 9am-5pm"],
)
print(result["score"])  # Low score -- hallucination detected

# Answer Relevancy (no retrieval_context needed)
relevancy = AnswerRelevancyEvaluator(model_config=config, threshold=0.7)
result = await relevancy.evaluate(
    input="What is the return policy?",
    actual_output="We offer 30-day returns at no extra cost.",
)

# Contextual Precision (ranking quality)
precision = ContextualPrecisionEvaluator(model_config=config, threshold=0.7)
result = await precision.evaluate(
    input="What is X?",
    actual_output="X is a programming concept.",
    expected_output="X is a well-known programming paradigm.",
    retrieval_context=["X is a programming paradigm.", "Unrelated info"],
)
```

#### RAG YAML Configuration

```
evaluations:
  model:
    provider: openai
    name: gpt-4o
  metrics:
    - type: rag
      metric_type: faithfulness
      threshold: 0.8
      include_reason: true

    - type: rag
      metric_type: answer_relevancy
      threshold: 0.7

    - type: rag
      metric_type: contextual_relevancy
      threshold: 0.6

    - type: rag
      metric_type: contextual_precision
      threshold: 0.7

    - type: rag
      metric_type: contextual_recall
      threshold: 0.6
```

#### RAG Metrics Summary

| Evaluator                      | Required Params                                                  | Requires `retrieval_context` | Measures                    |
| ------------------------------ | ---------------------------------------------------------------- | ---------------------------- | --------------------------- |
| `FaithfulnessEvaluator`        | `input`, `actual_output`, `retrieval_context`                    | Yes                          | Hallucination detection     |
| `AnswerRelevancyEvaluator`     | `input`, `actual_output`                                         | No                           | Response relevance to query |
| `ContextualRelevancyEvaluator` | `input`, `actual_output`, `retrieval_context`                    | Yes                          | Chunk relevance to query    |
| `ContextualPrecisionEvaluator` | `input`, `actual_output`, `expected_output`, `retrieval_context` | Yes                          | Ranking quality of chunks   |
| `ContextualRecallEvaluator`    | `input`, `actual_output`, `expected_output`, `retrieval_context` | Yes                          | Retrieval completeness      |

______________________________________________________________________

## Complete Agent Configuration Example

```
name: customer-support-agent
model:
  provider: openai
  name: gpt-4o

evaluations:
  model:
    provider: openai
    name: gpt-4o
    temperature: 0.0
  metrics:
    # Standard NLP metrics (no LLM required)
    - type: standard
      metric: bleu
      threshold: 0.4
    - type: standard
      metric: rouge
      threshold: 0.5

    # Custom G-Eval criteria
    - type: geval
      name: Helpfulness
      criteria: "Evaluate if the response provides actionable, helpful information"
      evaluation_params: [actual_output, input]
      threshold: 0.7

    # RAG evaluation
    - type: rag
      metric_type: faithfulness
      threshold: 0.8
      include_reason: true

test_cases:
  - name: "Refund policy question"
    input: "What is your refund policy?"
    ground_truth: "We offer a 30-day money-back guarantee on all products."
    retrieval_context:
      - "Refund Policy: All products come with a 30-day money-back guarantee."
      - "Returns must be initiated within 30 days of purchase."

  - name: "Product recommendation"
    input: "I need a laptop for video editing"
    expected_tools: [search_products, get_specifications]
    evaluations:
      - type: geval
        name: TechnicalAccuracy
        criteria: "Verify the response contains accurate technical specifications"
        threshold: 0.8
```

Run tests with:

```
holodeck test agent.yaml --verbose --output report.md --format markdown
```

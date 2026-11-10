# Contributing to HoloDeck

Start at [AGENTS.md](https://github.com/justinbarias/holodeck/blob/main/AGENTS.md) for repository instructions and the knowledge map. Read [architecture](https://github.com/justinbarias/holodeck/blob/main/ARCHITECTURE.md) before changing a module boundary.

## Development setup

The package targets Python 3.11+. CI tests Python 3.11 and 3.13; `make init` uses Python 3.11. Use UV to manage the environment and [uv.lock](https://github.com/justinbarias/holodeck/blob/main/uv.lock) to reproduce dependencies.

```
git clone https://github.com/justinbarias/holodeck.git
cd holodeck
uv sync --all-extras
make install-hooks
uv run holodeck --version
```

`uv run` selects the project environment without shell activation. `make init` also creates the environment, but requires a discoverable Python 3.11 executable. Use [pyproject.toml](https://github.com/justinbarias/holodeck/blob/main/pyproject.toml) for supported extras and dependency constraints. Use `uv add`, `uv add --dev`, or `uv remove` for requested dependency changes.

Environment priority is shell variables, project `.env`, then `~/.holodeck/.env`. Keep secrets out of version control and verification output. See [Security](https://docs.useholodeck.ai/SECURITY/index.md).

## Running tests

Run pytest with `-n auto`. Select the affected test area from [Quality score](https://docs.useholodeck.ai/QUALITY_SCORE/index.md).

```
uv run pytest tests/unit/lib/backends/test_selector.py -n auto -q
make test-unit-parallel
make test-integration-parallel
make test-parallel
uv run pytest -n auto --cov=src --cov-branch --cov-report=term-missing --cov-fail-under=80
```

The plain `make test`, `make test-unit`, and `make test-integration` targets currently run serially. The constitution sets an 80% coverage target. A focused test run does not establish repository-wide coverage. Integration and slow tests can require external services or credentials. Inspect their fixtures first. Use mocks for unit tests and committed fixtures for reproducible examples.

For bug fixes, first reproduce the failure with a test or an existing deterministic check. For behavior changes, cover meaningful edge cases and changed contracts. For documentation or formatting, use relevant checks without adding implementation-mirroring tests.

### Verify suite

Before a PR, run `make verify` (static gates, the full test suite, dependency audit, and an offline smoke of both backends). Run `make verify-live` when backends or serve change and you have credentials. [Reliability](https://docs.useholodeck.ai/RELIABILITY/#verify-suite) describes each tier, its proof, and the credentials it needs.

## Code style guide

Use Python 3.11-compatible syntax, Black at 88 columns, Ruff, and MyPy. [pyproject.toml](https://github.com/justinbarias/holodeck/blob/main/pyproject.toml) defines exact rules. Use Google-style docstrings for public modules, classes, and functions.

- Annotate functions and domain values with concrete types.
- Use protocols, discriminated unions, or `TypedDict` for known structures.
- Use `RunAgentInput` and `BaseEvent` for AG-UI request/event surfaces.
- Use backend result and event types at runtime boundaries.
- Convert untyped SDK values at the adapter boundary. Keep casts narrow.
- Do not access known model fields through `getattr` or arbitrary dictionaries.
- If dynamic SDK compatibility requires a helper, isolate it and document its typed return contract.
- If no practical alternative exists, use a specific `type: ignore` code and explain the reason.
- Use the [error hierarchy](https://github.com/justinbarias/holodeck/blob/main/src/holodeck/lib/errors.py) and preserve exception context.
- Catch specific exceptions. At process or protocol boundaries, convert failures to the documented error contract.
- Use Click output for CLI messages and logging for library diagnostics.
- Keep blocking I/O out of async execution. Use async clients or an appropriate thread boundary.
- Use Pydantic factories or `None` sentinels for mutable defaults.
- Justify new dependencies in the change description.

For touched Python files, run focused checks. Substitute the actual paths:

```
uv run black --target-version py311 --check scripts/check_harness.py tests/unit/test_harness.py
uv run ruff check scripts/check_harness.py tests/unit/test_harness.py
uv run mypy scripts/check_harness.py
```

`make format` mutates all source and test files. Prefer formatting touched files during scoped work. `make format-check`, `make lint`, and `make type-check` provide repository checks. `make lint` runs Ruff. `make security` separately runs pip-audit, Ruff security rules, Bandit, and detect-secrets. If a broad check fails for unrelated code or unavailable stubs, report that failure and the focused result.

## Documentation and schema checks

```
make harness-check
uv run mkdocs build
make schema
make schema-check
```

Run schema generation only for changes to the agent schema inputs. `make harness-check` also detects changes to the generated schema inventory. Use `make harness-generate` to refresh that inventory after schema changes.

The harness check covers mapped engineering documents and their local link targets. It does not crawl every historical spec, resolve Markdown heading anchors, or fetch external URLs. The docsite build can report separate historical link warnings. Record unresolved warnings as debt. Use `uv run mkdocs serve` for a local documentation preview.

## Pull request workflow

1. Define acceptance criteria and inspect the current working tree.
1. For complex work, maintain an [execution plan](https://docs.useholodeck.ai/PLANS/index.md).
1. Implement the scoped change and relevant regression coverage.
1. Run focused checks and review the final diff for unintended edits.
1. Update affected guides, schemas, design evidence, and spec status.
1. Report the change, checks, and any unresolved limitations.

Before committing, complete the constitution's formatting, type, test, and security checks. Run configured pre-commit hooks. Do not bypass required hooks to hide a failure. [CI](https://github.com/justinbarias/holodeck/blob/main/.github/workflows/ci.yml) runs hooks, parallel tests, and security scanning. `make ci` is a broad local workflow that also cleans caches and installs dependencies. It is not a required command after every small edit.

Use focused conventional commits, such as `fix(config): reject invalid provider`. Do not add Claude Code attribution or generated-by trailers. Stage only the intended files. Preserve unrelated user changes.

## Pre-commit hooks

```
make install-hooks
uv run pre-commit run --all-files
```

The [hook configuration](https://github.com/justinbarias/holodeck/blob/main/.pre-commit-config.yaml) is the source of truth for enabled checks. Security scanning also runs through `make security` in CI.

## Troubleshooting

For dependency errors, compare the installed environment with `uv.lock` before changing dependencies. For test failures, reproduce the smallest failing case with `-n auto` and inspect its fixtures. For runtime failures, use [Reliability](https://docs.useholodeck.ai/RELIABILITY/index.md) and capture redacted evidence. For architecture or test gaps, update [technical debt](https://docs.useholodeck.ai/exec-plans/tech-debt-tracker/index.md).

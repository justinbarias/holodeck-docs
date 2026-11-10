# Deployment Subsystem

The deployment subsystem packages HoloDeck agents as container images and deploys them to cloud providers. It covers three concerns: Dockerfile generation, Docker image building, and cloud deployment with state tracking.

Optional dependency

`ContainerBuilder` and `BuildResult` require the **docker** Python package. Install it with `pip install holodeck-ai[deploy]`. The symbols are lazily imported so the rest of the library works without Docker installed.

______________________________________________________________________

## Package entry point

## `deploy`

HoloDeck deployment engine.

This package provides the deployment functionality for HoloDeck agents, including Dockerfile generation, container building, and cloud deployment.

Note: ContainerBuilder and BuildResult require the 'docker' optional dependency (pip install holodeck-ai[deploy]). They are lazily imported to avoid ImportError when docker is not installed.

### `generate_dockerfile(agent_name, port, protocol, *, holodeck_version, holodeck_wheel=None, base_image='ghcr.io/justinbarias/holodeck-base:latest', version='0.0.0', source_url='', instruction_files=None, data_directories=None, environment=None, needs_nodejs=False, extras=None)`

Generate a Dockerfile for a HoloDeck agent.

The build context must hold `holodeck-constraints.txt` and `holodeck-overrides.txt` (the pin files shipped with the HoloDeck version being installed) and, for a development build, the wheel named by `holodeck_wheel`.

Parameters:

| Name                | Type             | Description                                                | Default                                                                                                                                                        |
| ------------------- | ---------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent_name`        | `str`            | Name of the agent for labeling                             | *required*                                                                                                                                                     |
| `port`              | `int`            | Port to expose                                             | *required*                                                                                                                                                     |
| `protocol`          | `str`            | Protocol type (rest, ag-ui, both)                          | *required*                                                                                                                                                     |
| `holodeck_version`  | `str`            | Exact HoloDeck version the image installs                  | *required*                                                                                                                                                     |
| `holodeck_wheel`    | \`str            | None\`                                                     | File name of a wheel in the build context. Set for a development version that is not on PyPI; the wheel is installed instead of holodeck-ai==holodeck_version. |
| `base_image`        | `str`            | Base Docker image to use                                   | `'ghcr.io/justinbarias/holodeck-base:latest'`                                                                                                                  |
| `version`           | `str`            | Version for OCI label                                      | `'0.0.0'`                                                                                                                                                      |
| `source_url`        | `str`            | Source URL for OCI label                                   | `''`                                                                                                                                                           |
| `instruction_files` | \`list[str]      | None\`                                                     | List of instruction file paths to copy                                                                                                                         |
| `data_directories`  | \`list[str]      | None\`                                                     | List of data directories to copy                                                                                                                               |
| `environment`       | \`dict[str, str] | None\`                                                     | Environment variables to set                                                                                                                                   |
| `needs_nodejs`      | `bool`           | Whether to install Node.js (required for Claude Agent SDK) | `False`                                                                                                                                                        |
| `extras`            | \`list[str]      | None\`                                                     | holodeck-ai extras to install (e.g., ["chromadb"]); rendered sorted and without duplicates                                                                     |

Returns:

| Type  | Description                              |
| ----- | ---------------------------------------- |
| `str` | Generated Dockerfile content as a string |

Raises:

| Type         | Description                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------ |
| `ValueError` | If the version, wheel name, or an extra name has characters that do not belong in a requirement. |

Example

> > > dockerfile = generate_dockerfile( ... agent_name="my-agent", ... port=8080, ... protocol="rest", ... holodeck_version="1.2.3", ... instruction_files=["instructions.md"], ... ) print(dockerfile[:50])

#### HoloDeck Agent Container

#### Auto-generated Doc...

Source code in `src/holodeck/deploy/dockerfile.py`

```
def generate_dockerfile(
    agent_name: str,
    port: int,
    protocol: str,
    *,
    holodeck_version: str,
    holodeck_wheel: str | None = None,
    base_image: str = "ghcr.io/justinbarias/holodeck-base:latest",
    version: str = "0.0.0",
    source_url: str = "",
    instruction_files: list[str] | None = None,
    data_directories: list[str] | None = None,
    environment: dict[str, str] | None = None,
    needs_nodejs: bool = False,
    extras: list[str] | None = None,
) -> str:
    """Generate a Dockerfile for a HoloDeck agent.

    The build context must hold ``holodeck-constraints.txt`` and
    ``holodeck-overrides.txt`` (the pin files shipped with the HoloDeck version
    being installed) and, for a development build, the wheel named by
    ``holodeck_wheel``.

    Args:
        agent_name: Name of the agent for labeling
        port: Port to expose
        protocol: Protocol type (rest, ag-ui, both)
        holodeck_version: Exact HoloDeck version the image installs
        holodeck_wheel: File name of a wheel in the build context. Set for a
            development version that is not on PyPI; the wheel is installed
            instead of ``holodeck-ai==holodeck_version``.
        base_image: Base Docker image to use
        version: Version for OCI label
        source_url: Source URL for OCI label
        instruction_files: List of instruction file paths to copy
        data_directories: List of data directories to copy
        environment: Environment variables to set
        needs_nodejs: Whether to install Node.js (required for Claude Agent SDK)
        extras: holodeck-ai extras to install (e.g., ["chromadb"]); rendered
            sorted and without duplicates

    Returns:
        Generated Dockerfile content as a string

    Raises:
        ValueError: If the version, wheel name, or an extra name has characters
            that do not belong in a requirement.

    Example:
        >>> dockerfile = generate_dockerfile(
        ...     agent_name="my-agent",
        ...     port=8080,
        ...     protocol="rest",
        ...     holodeck_version="1.2.3",
        ...     instruction_files=["instructions.md"],
        ... )
        >>> print(dockerfile[:50])
        # HoloDeck Agent Container
        # Auto-generated Doc...
    """
    extras_list = sorted(set(extras or []))
    _validate_install(holodeck_version, holodeck_wheel, extras_list)

    # jinja2 types Template.__new__ as returning Any; annotate to keep render() typed.
    template: Template = Template(HOLODECK_DOCKERFILE_TEMPLATE)

    # Generate ISO 8601 timestamp
    created = datetime.now(UTC).isoformat()

    return template.render(
        agent_name=agent_name,
        port=port,
        protocol=protocol,
        base_image=base_image,
        version=version,
        source_url=source_url,
        created=created,
        instruction_files=instruction_files or [],
        data_directories=data_directories or [],
        environment=environment or {},
        needs_nodejs=needs_nodejs,
        extras=extras_list,
        holodeck_version=holodeck_version,
        holodeck_wheel=holodeck_wheel,
        requirement=_requirement(holodeck_version, holodeck_wheel, extras_list),
        manifest_header=(
            f"# holodeck-ai {holodeck_version}; extras: "
            f"{','.join(extras_list) or 'none'}"
        ),
        constraints_file=CONSTRAINTS_CONTEXT_FILE,
        overrides_file=OVERRIDES_CONTEXT_FILE,
        manifest_path=MANIFEST_IMAGE_PATH,
    )
```

### `generate_tag(strategy, custom_tag=None)`

Generate an image tag based on the specified strategy.

Parameters:

| Name         | Type          | Description                                                | Default                                  |
| ------------ | ------------- | ---------------------------------------------------------- | ---------------------------------------- |
| `strategy`   | `TagStrategy` | Tag generation strategy (git_sha, git_tag, latest, custom) | *required*                               |
| `custom_tag` | \`str         | None\`                                                     | Custom tag value when strategy is CUSTOM |

Returns:

| Type  | Description          |
| ----- | -------------------- |
| `str` | Generated tag string |

Raises:

| Type              | Description                                             |
| ----------------- | ------------------------------------------------------- |
| `ValueError`      | If custom strategy is used without providing custom_tag |
| `DeploymentError` | If git commands fail (not in repo, no tags, etc.)       |

Example

> > > generate_tag(TagStrategy.LATEST) 'latest' generate_tag(TagStrategy.CUSTOM, custom_tag="v1.0.0") 'v1.0.0'

Source code in `src/holodeck/deploy/builder.py`

```
def generate_tag(strategy: TagStrategy, custom_tag: str | None = None) -> str:
    """Generate an image tag based on the specified strategy.

    Args:
        strategy: Tag generation strategy (git_sha, git_tag, latest, custom)
        custom_tag: Custom tag value when strategy is CUSTOM

    Returns:
        Generated tag string

    Raises:
        ValueError: If custom strategy is used without providing custom_tag
        DeploymentError: If git commands fail (not in repo, no tags, etc.)

    Example:
        >>> generate_tag(TagStrategy.LATEST)
        'latest'
        >>> generate_tag(TagStrategy.CUSTOM, custom_tag="v1.0.0")
        'v1.0.0'
    """
    if strategy == TagStrategy.LATEST:
        return "latest"

    if strategy == TagStrategy.CUSTOM:
        if not custom_tag:
            raise ValueError("custom_tag is required when using CUSTOM strategy")
        return custom_tag

    if strategy == TagStrategy.GIT_SHA:
        result = subprocess.run(  # noqa: S603  # nosec B603 B607
            ["git", "rev-parse", "HEAD"],  # noqa: S607
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            raise DeploymentError(
                operation="tag_generation",
                message="Failed to get git SHA: not a git repository",
            )
        # Return first 7 characters of SHA
        return result.stdout.strip()[:7]

    if strategy == TagStrategy.GIT_TAG:
        result = subprocess.run(  # noqa: S603  # nosec B603 B607
            ["git", "describe", "--tags", "--abbrev=0"],  # noqa: S607
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            raise DeploymentError(
                operation="tag_generation",
                message="No git tags found. Create a tag first: git tag v1.0.0",
            )
        return result.stdout.strip()

    # Should not reach here, but handle gracefully
    raise ValueError(f"Unknown tag strategy: {strategy}")
```

### `get_oci_labels(agent_name, version, source_sha=None)`

Generate OCI-compliant container image labels.

Parameters:

| Name         | Type  | Description                       | Default                              |
| ------------ | ----- | --------------------------------- | ------------------------------------ |
| `agent_name` | `str` | Name of the agent for image title | *required*                           |
| `version`    | `str` | Version string for the image      | *required*                           |
| `source_sha` | \`str | None\`                            | Optional git SHA for source tracking |

Returns:

| Type             | Description              |
| ---------------- | ------------------------ |
| `dict[str, str]` | Dictionary of OCI labels |

Example

> > > labels = get_oci_labels("my-agent", "v1.0.0") labels["org.opencontainers.image.title"] 'my-agent'

Source code in `src/holodeck/deploy/builder.py`

```
def get_oci_labels(
    agent_name: str,
    version: str,
    source_sha: str | None = None,
) -> dict[str, str]:
    """Generate OCI-compliant container image labels.

    Args:
        agent_name: Name of the agent for image title
        version: Version string for the image
        source_sha: Optional git SHA for source tracking

    Returns:
        Dictionary of OCI labels

    Example:
        >>> labels = get_oci_labels("my-agent", "v1.0.0")
        >>> labels["org.opencontainers.image.title"]
        'my-agent'
    """
    created = datetime.now(UTC).isoformat()

    labels = {
        "org.opencontainers.image.title": agent_name,
        "org.opencontainers.image.version": version,
        "org.opencontainers.image.created": created,
        "com.holodeck.managed": "true",
    }

    if source_sha:
        # Use first 7 characters of SHA
        labels["org.opencontainers.image.source"] = source_sha[:7]

    return labels
```

______________________________________________________________________

## `holodeck.deploy.dockerfile` -- Dockerfile generation

Generates Dockerfiles from Jinja2 templates, embedding OCI labels, environment variables, instruction files, and optional Node.js installation for Claude agents.

## `generate_dockerfile(agent_name, port, protocol, *, holodeck_version, holodeck_wheel=None, base_image='ghcr.io/justinbarias/holodeck-base:latest', version='0.0.0', source_url='', instruction_files=None, data_directories=None, environment=None, needs_nodejs=False, extras=None)`

Generate a Dockerfile for a HoloDeck agent.

The build context must hold `holodeck-constraints.txt` and `holodeck-overrides.txt` (the pin files shipped with the HoloDeck version being installed) and, for a development build, the wheel named by `holodeck_wheel`.

Parameters:

| Name                | Type             | Description                                                | Default                                                                                                                                                        |
| ------------------- | ---------------- | ---------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agent_name`        | `str`            | Name of the agent for labeling                             | *required*                                                                                                                                                     |
| `port`              | `int`            | Port to expose                                             | *required*                                                                                                                                                     |
| `protocol`          | `str`            | Protocol type (rest, ag-ui, both)                          | *required*                                                                                                                                                     |
| `holodeck_version`  | `str`            | Exact HoloDeck version the image installs                  | *required*                                                                                                                                                     |
| `holodeck_wheel`    | \`str            | None\`                                                     | File name of a wheel in the build context. Set for a development version that is not on PyPI; the wheel is installed instead of holodeck-ai==holodeck_version. |
| `base_image`        | `str`            | Base Docker image to use                                   | `'ghcr.io/justinbarias/holodeck-base:latest'`                                                                                                                  |
| `version`           | `str`            | Version for OCI label                                      | `'0.0.0'`                                                                                                                                                      |
| `source_url`        | `str`            | Source URL for OCI label                                   | `''`                                                                                                                                                           |
| `instruction_files` | \`list[str]      | None\`                                                     | List of instruction file paths to copy                                                                                                                         |
| `data_directories`  | \`list[str]      | None\`                                                     | List of data directories to copy                                                                                                                               |
| `environment`       | \`dict[str, str] | None\`                                                     | Environment variables to set                                                                                                                                   |
| `needs_nodejs`      | `bool`           | Whether to install Node.js (required for Claude Agent SDK) | `False`                                                                                                                                                        |
| `extras`            | \`list[str]      | None\`                                                     | holodeck-ai extras to install (e.g., ["chromadb"]); rendered sorted and without duplicates                                                                     |

Returns:

| Type  | Description                              |
| ----- | ---------------------------------------- |
| `str` | Generated Dockerfile content as a string |

Raises:

| Type         | Description                                                                                      |
| ------------ | ------------------------------------------------------------------------------------------------ |
| `ValueError` | If the version, wheel name, or an extra name has characters that do not belong in a requirement. |

Example

> > > dockerfile = generate_dockerfile( ... agent_name="my-agent", ... port=8080, ... protocol="rest", ... holodeck_version="1.2.3", ... instruction_files=["instructions.md"], ... ) print(dockerfile[:50])

### HoloDeck Agent Container

### Auto-generated Doc...

Source code in `src/holodeck/deploy/dockerfile.py`

```
def generate_dockerfile(
    agent_name: str,
    port: int,
    protocol: str,
    *,
    holodeck_version: str,
    holodeck_wheel: str | None = None,
    base_image: str = "ghcr.io/justinbarias/holodeck-base:latest",
    version: str = "0.0.0",
    source_url: str = "",
    instruction_files: list[str] | None = None,
    data_directories: list[str] | None = None,
    environment: dict[str, str] | None = None,
    needs_nodejs: bool = False,
    extras: list[str] | None = None,
) -> str:
    """Generate a Dockerfile for a HoloDeck agent.

    The build context must hold ``holodeck-constraints.txt`` and
    ``holodeck-overrides.txt`` (the pin files shipped with the HoloDeck version
    being installed) and, for a development build, the wheel named by
    ``holodeck_wheel``.

    Args:
        agent_name: Name of the agent for labeling
        port: Port to expose
        protocol: Protocol type (rest, ag-ui, both)
        holodeck_version: Exact HoloDeck version the image installs
        holodeck_wheel: File name of a wheel in the build context. Set for a
            development version that is not on PyPI; the wheel is installed
            instead of ``holodeck-ai==holodeck_version``.
        base_image: Base Docker image to use
        version: Version for OCI label
        source_url: Source URL for OCI label
        instruction_files: List of instruction file paths to copy
        data_directories: List of data directories to copy
        environment: Environment variables to set
        needs_nodejs: Whether to install Node.js (required for Claude Agent SDK)
        extras: holodeck-ai extras to install (e.g., ["chromadb"]); rendered
            sorted and without duplicates

    Returns:
        Generated Dockerfile content as a string

    Raises:
        ValueError: If the version, wheel name, or an extra name has characters
            that do not belong in a requirement.

    Example:
        >>> dockerfile = generate_dockerfile(
        ...     agent_name="my-agent",
        ...     port=8080,
        ...     protocol="rest",
        ...     holodeck_version="1.2.3",
        ...     instruction_files=["instructions.md"],
        ... )
        >>> print(dockerfile[:50])
        # HoloDeck Agent Container
        # Auto-generated Doc...
    """
    extras_list = sorted(set(extras or []))
    _validate_install(holodeck_version, holodeck_wheel, extras_list)

    # jinja2 types Template.__new__ as returning Any; annotate to keep render() typed.
    template: Template = Template(HOLODECK_DOCKERFILE_TEMPLATE)

    # Generate ISO 8601 timestamp
    created = datetime.now(UTC).isoformat()

    return template.render(
        agent_name=agent_name,
        port=port,
        protocol=protocol,
        base_image=base_image,
        version=version,
        source_url=source_url,
        created=created,
        instruction_files=instruction_files or [],
        data_directories=data_directories or [],
        environment=environment or {},
        needs_nodejs=needs_nodejs,
        extras=extras_list,
        holodeck_version=holodeck_version,
        holodeck_wheel=holodeck_wheel,
        requirement=_requirement(holodeck_version, holodeck_wheel, extras_list),
        manifest_header=(
            f"# holodeck-ai {holodeck_version}; extras: "
            f"{','.join(extras_list) or 'none'}"
        ),
        constraints_file=CONSTRAINTS_CONTEXT_FILE,
        overrides_file=OVERRIDES_CONTEXT_FILE,
        manifest_path=MANIFEST_IMAGE_PATH,
    )
```

______________________________________________________________________

## `holodeck.deploy.extras` -- Extras for a serving image

Chooses the `holodeck-ai` extras an agent needs. Provider extras come from the retrieval registry.

## `required_extras(agent, *, temporal_worker=False)`

Return the `holodeck-ai` extras that `agent` needs to serve.

Parameters:

| Name              | Type    | Description                                                                                                                           | Default    |
| ----------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------- | ---------- |
| `agent`           | `Agent` | Loaded agent configuration.                                                                                                           | *required* |
| `temporal_worker` | `bool`  | The image runs holodeck worker (Temporal), which needs the temporal extra. Nothing in an agent file says this, so the caller decides. | `False`    |

Returns:

| Type        | Description                                                           |
| ----------- | --------------------------------------------------------------------- |
| `list[str]` | Extra names, sorted and without duplicates. A database that the agent |
| `list[str]` | names by reference and the loader could not resolve adds nothing.     |

Source code in `src/holodeck/deploy/extras.py`

```
def required_extras(agent: Agent, *, temporal_worker: bool = False) -> list[str]:
    """Return the ``holodeck-ai`` extras that ``agent`` needs to serve.

    Args:
        agent: Loaded agent configuration.
        temporal_worker: The image runs ``holodeck worker`` (Temporal), which
            needs the ``temporal`` extra. Nothing in an agent file says this,
            so the caller decides.

    Returns:
        Extra names, sorted and without duplicates. A database that the agent
        names by reference and the loader could not resolve adds nothing.
    """
    extras: set[str] = set()

    provider = agent.model.provider
    if provider in _OPENAI_AGENTS_PROVIDERS:
        extras.add("openai-agents")

    observability = agent.observability
    if observability is not None and observability.enabled:
        if provider in _CLAUDE_BACKEND_PROVIDERS and observability.traces.enabled:
            extras.add("claude-otel")
        exporters = observability.exporters
        if exporters.prometheus and exporters.prometheus.enabled:
            extras.add("otel-prometheus")
        if exporters.azure_monitor and exporters.azure_monitor.enabled:
            extras.add("otel-azmon")

    for tool in agent.tools or []:
        if not isinstance(tool, VectorstoreTool | HierarchicalDocumentToolConfig):
            continue
        if isinstance(tool.database, DatabaseConfig):
            entry = PROVIDERS.get(tool.database.provider)
            if entry is not None and entry.extra is not None:
                extras.add(entry.extra)
        for scheme, extra in _SOURCE_SCHEME_EXTRAS.items():
            if tool.source.startswith(scheme):
                extras.add(extra)
        if isinstance(tool, HierarchicalDocumentToolConfig):
            keyword_index = tool.keyword_index
            if keyword_index is not None and keyword_index.provider == "opensearch":
                extras.add("opensearch")

    if temporal_worker:
        extras.add("temporal")

    return sorted(extras)
```

______________________________________________________________________

## `holodeck.deploy.install` -- Which HoloDeck an image installs

An image installs one exact version: a PyPI release, or a wheel for a development version. The constraints and overrides files come from the package being installed.

## `resolve_install(installed_version, wheel=None, search_dirs=None)`

Choose the HoloDeck an image installs.

Parameters:

| Name                | Type         | Description                      | Default                                                                                                                                      |
| ------------------- | ------------ | -------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| `installed_version` | `str`        | Version of the running HoloDeck. | *required*                                                                                                                                   |
| `wheel`             | \`Path       | None\`                           | Wheel passed by the user. It wins over everything else and may be any version.                                                               |
| `search_dirs`       | \`list[Path] | None\`                           | Where to look for holodeck_ai-<version>-\*.whl when the running version is a development version and no wheel is given (for example ./dist). |

Raises:

| Type              | Description                                                                          |
| ----------------- | ------------------------------------------------------------------------------------ |
| `DeploymentError` | A development version has no wheel, or a given wheel is not a usable HoloDeck wheel. |

Source code in `src/holodeck/deploy/install.py`

```
def resolve_install(
    installed_version: str,
    wheel: Path | None = None,
    search_dirs: list[Path] | None = None,
) -> HoloDeckInstall:
    """Choose the HoloDeck an image installs.

    Args:
        installed_version: Version of the running HoloDeck.
        wheel: Wheel passed by the user. It wins over everything else and may
            be any version.
        search_dirs: Where to look for ``holodeck_ai-<version>-*.whl`` when
            the running version is a development version and no wheel is
            given (for example ``./dist``).

    Raises:
        DeploymentError: A development version has no wheel, or a given wheel
            is not a usable HoloDeck wheel.
    """
    if wheel is not None:
        if not wheel.is_file():
            raise DeploymentError(
                operation="build", message=f"Wheel not found: {wheel}"
            )
        return HoloDeckInstall(
            version=wheel_version(wheel),
            constraints=wheel_constraints(wheel),
            overrides=wheel_overrides(wheel),
            wheel=wheel,
        )

    if is_published_version(installed_version):
        return HoloDeckInstall(
            version=installed_version,
            constraints=bundled_constraints(),
            overrides=bundled_overrides(),
        )

    found = _find_dev_wheel(installed_version, search_dirs or [])
    if found is not None:
        return HoloDeckInstall(
            version=wheel_version(found),
            constraints=wheel_constraints(found),
            overrides=wheel_overrides(found),
            wheel=found,
        )
    raise DeploymentError(
        operation="build",
        message=(
            f"HoloDeck {installed_version} is a development version and is not "
            "on PyPI, so an image cannot install it by version. Build a wheel "
            "with `uv build --wheel` and pass it with `--wheel dist/<file>.whl`, "
            "or run `holodeck deploy build` from a released HoloDeck."
        ),
    )
```

______________________________________________________________________

## `holodeck.deploy.manifest` -- Image manifest

The manifest `holodeck deploy build` writes to `.holodeck/builds/<tag>.json`.

## `build_manifest(*, agent, platform, install, extras, base_image, image_reference, image, freeze_text)`

Assemble the manifest of one build.

Raises:

| Type              | Description                                                                                                       |
| ----------------- | ----------------------------------------------------------------------------------------------------------------- |
| `DeploymentError` | If the image reported no packages. An install that produced an empty list did not run, so the manifest would lie. |

Source code in `src/holodeck/deploy/manifest.py`

```
def build_manifest(
    *,
    agent: str,
    platform: str,
    install: HoloDeckInstall,
    extras: list[str],
    base_image: ImageRef,
    image_reference: str,
    image: ImageRef,
    freeze_text: str,
) -> ImageManifest:
    """Assemble the manifest of one build.

    Raises:
        DeploymentError: If the image reported no packages. An install that
            produced an empty list did not run, so the manifest would lie.
    """
    packages = parse_freeze(freeze_text)
    if not packages:
        raise DeploymentError(
            operation="build",
            message="The image package manifest is empty; the install did not run.",
        )
    return ImageManifest(
        agent=agent,
        platform=platform,
        holodeck=HoloDeckRecord(
            version=install.version,
            source=install.source,
            wheel_sha256=(
                _sha256_file(install.wheel) if install.wheel is not None else None
            ),
            extras=sorted(set(extras)),
            constraints_sha256=sha256_text(install.constraints),
            overrides_sha256=sha256_text(install.overrides),
        ),
        base_image=BaseImageRecord(
            reference=base_image.reference,
            digest=base_image.digest,
            image_id=base_image.image_id,
        ),
        image=ImageRecord(
            reference=image_reference,
            image_id=image.image_id,
            digest=image.digest,
        ),
        packages=packages,
    )
```

______________________________________________________________________

## `holodeck.deploy.builder` -- Container image building

Builds Docker images via the Docker SDK and provides helpers for tag generation and OCI label creation.

### BuildResult

## `BuildResult(image_id, image_name, tag, full_name, log_lines=list())`

Result of a container image build operation.

Attributes:

| Name         | Type        | Description                      |
| ------------ | ----------- | -------------------------------- |
| `image_id`   | `str`       | The SHA256 ID of the built image |
| `image_name` | `str`       | The repository/image name        |
| `tag`        | `str`       | The image tag                    |
| `full_name`  | `str`       | Full image reference (name:tag)  |
| `log_lines`  | `list[str]` | Build log output lines           |

### `from_image(image, image_name, tag, log_lines=None)`

Create BuildResult from a Docker image object.

Parameters:

| Name         | Type        | Description                    | Default                  |
| ------------ | ----------- | ------------------------------ | ------------------------ |
| `image`      | `Image`     | Docker image object from build | *required*               |
| `image_name` | `str`       | Repository/image name          | *required*               |
| `tag`        | `str`       | Image tag                      | *required*               |
| `log_lines`  | \`list[str] | None\`                         | Optional build log lines |

Returns:

| Type          | Description          |
| ------------- | -------------------- |
| `BuildResult` | BuildResult instance |

Source code in `src/holodeck/deploy/builder.py`

```
@classmethod
def from_image(
    cls,
    image: Image,
    image_name: str,
    tag: str,
    log_lines: list[str] | None = None,
) -> BuildResult:
    """Create BuildResult from a Docker image object.

    Args:
        image: Docker image object from build
        image_name: Repository/image name
        tag: Image tag
        log_lines: Optional build log lines

    Returns:
        BuildResult instance
    """
    image_id = image.id or ""
    return cls(
        image_id=image_id,
        image_name=image_name,
        tag=tag,
        full_name=f"{image_name}:{tag}",
        log_lines=log_lines or [],
    )
```

### ContainerBuilder

## `ContainerBuilder()`

Builder for HoloDeck agent container images.

Uses the Docker SDK to build container images from agent configurations. Handles Docker daemon connection, build execution, and error handling.

Example

> > > builder = ContainerBuilder() result = builder.build( ... build_context="./build", ... image_name="my-org/my-agent", ... tag="v1.0.0", ... ) print(result.full_name) 'my-org/my-agent:v1.0.0'

Initialize the container builder.

Connects to the Docker daemon using the environment configuration.

Raises:

| Type                      | Description                       |
| ------------------------- | --------------------------------- |
| `DockerNotAvailableError` | If Docker daemon is not available |

Source code in `src/holodeck/deploy/builder.py`

```
def __init__(self) -> None:
    """Initialize the container builder.

    Connects to the Docker daemon using the environment configuration.

    Raises:
        DockerNotAvailableError: If Docker daemon is not available
    """
    try:
        import docker
        from docker.errors import DockerException

        self.client = docker.from_env()  # type: ignore[attr-defined]
    except DockerException as e:
        raise DockerNotAvailableError(operation="init") from e
```

### `build(build_context, image_name, tag, labels=None, dockerfile='Dockerfile', platform='linux/amd64', pull=True, **build_kwargs)`

Build a container image from the specified context.

Parameters:

| Name             | Type             | Description                                                                                                                                                        | Default                      |
| ---------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------- |
| `build_context`  | `str`            | Path to the build context directory                                                                                                                                | *required*                   |
| `image_name`     | `str`            | Repository/image name for the built image                                                                                                                          | *required*                   |
| `tag`            | `str`            | Tag for the built image                                                                                                                                            | *required*                   |
| `labels`         | \`dict[str, str] | None\`                                                                                                                                                             | Optional OCI labels to apply |
| `dockerfile`     | `str`            | Path to Dockerfile relative to context                                                                                                                             | `'Dockerfile'`               |
| `platform`       | `str`            | Target platform for the image (default: linux/amd64)                                                                                                               | `'linux/amd64'`              |
| `pull`           | `bool`           | Pull the base image first (default). False builds on the base image already in the local daemon, for a base image that was built locally and is not in a registry. | `True`                       |
| `**build_kwargs` | `Any`            | Additional arguments passed to Docker build                                                                                                                        | `{}`                         |

Returns:

| Type          | Description                                   |
| ------------- | --------------------------------------------- |
| `BuildResult` | BuildResult with image details and build logs |

Raises:

| Type              | Description                                   |
| ----------------- | --------------------------------------------- |
| `DeploymentError` | If build context doesn't exist or build fails |

Source code in `src/holodeck/deploy/builder.py`

```
def build(
    self,
    build_context: str,
    image_name: str,
    tag: str,
    labels: dict[str, str] | None = None,
    dockerfile: str = "Dockerfile",
    platform: str = "linux/amd64",
    pull: bool = True,
    **build_kwargs: Any,
) -> BuildResult:
    """Build a container image from the specified context.

    Args:
        build_context: Path to the build context directory
        image_name: Repository/image name for the built image
        tag: Tag for the built image
        labels: Optional OCI labels to apply
        dockerfile: Path to Dockerfile relative to context
        platform: Target platform for the image (default: linux/amd64)
        pull: Pull the base image first (default). False builds on the
            base image already in the local daemon, for a base image that
            was built locally and is not in a registry.
        **build_kwargs: Additional arguments passed to Docker build

    Returns:
        BuildResult with image details and build logs

    Raises:
        DeploymentError: If build context doesn't exist or build fails
    """
    context_path = Path(build_context)
    if not context_path.exists():
        raise DeploymentError(
            operation="build",
            message=f"Build context not found: {build_context}",
        )

    full_tag = f"{image_name}:{tag}"

    try:
        image, build_logs = self.client.images.build(
            path=str(context_path),
            tag=full_tag,
            dockerfile=dockerfile,
            labels=labels or {},
            rm=True,  # Remove intermediate containers
            platform=platform,
            pull=pull,  # Pull the base image unless the caller opts out
            **build_kwargs,
        )

        # Extract log lines from build output
        log_lines: list[str] = []
        for log_entry in build_logs:
            # Docker SDK returns dict[str, Any] for log entries
            if isinstance(log_entry, dict):
                if "stream" in log_entry:
                    stream_val = log_entry["stream"]
                    if isinstance(stream_val, str):
                        log_lines.append(stream_val.rstrip("\n"))
                elif "error" in log_entry:
                    log_lines.append(f"ERROR: {log_entry['error']}")

        return BuildResult.from_image(
            image=image,
            image_name=image_name,
            tag=tag,
            log_lines=log_lines,
        )

    except Exception as e:
        # docker is guaranteed imported since __init__ succeeded
        from docker.errors import BuildError, DockerException

        if isinstance(e, BuildError):
            raise DeploymentError(
                operation="build",
                message=f"Docker build failed: {e.msg}",
            ) from e
        elif isinstance(e, DockerException):
            raise DeploymentError(
                operation="build",
                message=f"Docker error during build: {e}",
            ) from e
        raise
```

### generate_tag

## `generate_tag(strategy, custom_tag=None)`

Generate an image tag based on the specified strategy.

Parameters:

| Name         | Type          | Description                                                | Default                                  |
| ------------ | ------------- | ---------------------------------------------------------- | ---------------------------------------- |
| `strategy`   | `TagStrategy` | Tag generation strategy (git_sha, git_tag, latest, custom) | *required*                               |
| `custom_tag` | \`str         | None\`                                                     | Custom tag value when strategy is CUSTOM |

Returns:

| Type  | Description          |
| ----- | -------------------- |
| `str` | Generated tag string |

Raises:

| Type              | Description                                             |
| ----------------- | ------------------------------------------------------- |
| `ValueError`      | If custom strategy is used without providing custom_tag |
| `DeploymentError` | If git commands fail (not in repo, no tags, etc.)       |

Example

> > > generate_tag(TagStrategy.LATEST) 'latest' generate_tag(TagStrategy.CUSTOM, custom_tag="v1.0.0") 'v1.0.0'

Source code in `src/holodeck/deploy/builder.py`

```
def generate_tag(strategy: TagStrategy, custom_tag: str | None = None) -> str:
    """Generate an image tag based on the specified strategy.

    Args:
        strategy: Tag generation strategy (git_sha, git_tag, latest, custom)
        custom_tag: Custom tag value when strategy is CUSTOM

    Returns:
        Generated tag string

    Raises:
        ValueError: If custom strategy is used without providing custom_tag
        DeploymentError: If git commands fail (not in repo, no tags, etc.)

    Example:
        >>> generate_tag(TagStrategy.LATEST)
        'latest'
        >>> generate_tag(TagStrategy.CUSTOM, custom_tag="v1.0.0")
        'v1.0.0'
    """
    if strategy == TagStrategy.LATEST:
        return "latest"

    if strategy == TagStrategy.CUSTOM:
        if not custom_tag:
            raise ValueError("custom_tag is required when using CUSTOM strategy")
        return custom_tag

    if strategy == TagStrategy.GIT_SHA:
        result = subprocess.run(  # noqa: S603  # nosec B603 B607
            ["git", "rev-parse", "HEAD"],  # noqa: S607
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            raise DeploymentError(
                operation="tag_generation",
                message="Failed to get git SHA: not a git repository",
            )
        # Return first 7 characters of SHA
        return result.stdout.strip()[:7]

    if strategy == TagStrategy.GIT_TAG:
        result = subprocess.run(  # noqa: S603  # nosec B603 B607
            ["git", "describe", "--tags", "--abbrev=0"],  # noqa: S607
            capture_output=True,
            text=True,
        )
        if result.returncode != 0:
            raise DeploymentError(
                operation="tag_generation",
                message="No git tags found. Create a tag first: git tag v1.0.0",
            )
        return result.stdout.strip()

    # Should not reach here, but handle gracefully
    raise ValueError(f"Unknown tag strategy: {strategy}")
```

### get_oci_labels

## `get_oci_labels(agent_name, version, source_sha=None)`

Generate OCI-compliant container image labels.

Parameters:

| Name         | Type  | Description                       | Default                              |
| ------------ | ----- | --------------------------------- | ------------------------------------ |
| `agent_name` | `str` | Name of the agent for image title | *required*                           |
| `version`    | `str` | Version string for the image      | *required*                           |
| `source_sha` | \`str | None\`                            | Optional git SHA for source tracking |

Returns:

| Type             | Description              |
| ---------------- | ------------------------ |
| `dict[str, str]` | Dictionary of OCI labels |

Example

> > > labels = get_oci_labels("my-agent", "v1.0.0") labels["org.opencontainers.image.title"] 'my-agent'

Source code in `src/holodeck/deploy/builder.py`

```
def get_oci_labels(
    agent_name: str,
    version: str,
    source_sha: str | None = None,
) -> dict[str, str]:
    """Generate OCI-compliant container image labels.

    Args:
        agent_name: Name of the agent for image title
        version: Version string for the image
        source_sha: Optional git SHA for source tracking

    Returns:
        Dictionary of OCI labels

    Example:
        >>> labels = get_oci_labels("my-agent", "v1.0.0")
        >>> labels["org.opencontainers.image.title"]
        'my-agent'
    """
    created = datetime.now(UTC).isoformat()

    labels = {
        "org.opencontainers.image.title": agent_name,
        "org.opencontainers.image.version": version,
        "org.opencontainers.image.created": created,
        "com.holodeck.managed": "true",
    }

    if source_sha:
        # Use first 7 characters of SHA
        labels["org.opencontainers.image.source"] = source_sha[:7]

    return labels
```

______________________________________________________________________

## `holodeck.deploy.state` -- Deployment state tracking

Persists deployment records to a JSON file under `.holodeck/deployments.json` next to the agent configuration. Tracks creation and update timestamps and computes deterministic config hashes for drift detection.

### get_state_path

## `get_state_path(agent_path)`

Return the deployment state file path for an agent config.

Source code in `src/holodeck/deploy/state.py`

```
def get_state_path(agent_path: Path) -> Path:
    """Return the deployment state file path for an agent config."""
    return agent_path.parent / ".holodeck" / "deployments.json"
```

### compute_config_hash

## `compute_config_hash(config)`

Compute a deterministic hash for the deployment configuration.

Source code in `src/holodeck/deploy/state.py`

```
def compute_config_hash(config: DeploymentConfig) -> str:
    """Compute a deterministic hash for the deployment configuration."""
    payload = json.dumps(
        config.model_dump(mode="json", exclude_unset=True),
        sort_keys=True,
    )
    digest = hashlib.sha256(payload.encode("utf-8")).hexdigest()
    return f"sha256:{digest}"
```

### load_state

## `load_state(state_path)`

Load deployment state data from disk.

Source code in `src/holodeck/deploy/state.py`

```
def load_state(state_path: Path) -> DeploymentState:
    """Load deployment state data from disk."""
    if not state_path.exists():
        return DeploymentState(version=STATE_VERSION)

    try:
        content = state_path.read_text(encoding="utf-8")
        if not content.strip():
            return DeploymentState(version=STATE_VERSION)
    except OSError as exc:
        raise DeploymentError(
            operation="state",
            message=f"Failed to read deployment state at {state_path}: {exc}",
        ) from exc

    try:
        state = DeploymentState.model_validate_json(content)
    except ValidationError as exc:
        raise DeploymentError(
            operation="state",
            message=f"Invalid deployment state format in {state_path}: {exc}",
        ) from exc

    if not state.version:
        state = state.model_copy(update={"version": STATE_VERSION})
    return state
```

### save_state

## `save_state(state_path, state)`

Persist deployment state data to disk.

Source code in `src/holodeck/deploy/state.py`

```
def save_state(state_path: Path, state: DeploymentState) -> None:
    """Persist deployment state data to disk."""
    try:
        state_path.parent.mkdir(parents=True, exist_ok=True)
        payload = json.dumps(state.model_dump(mode="json"), indent=2, sort_keys=True)
        state_path.write_text(payload, encoding="utf-8")
    except OSError as exc:
        raise DeploymentError(
            operation="state",
            message=f"Failed to write deployment state to {state_path}: {exc}",
        ) from exc
```

### get_deployment_record

## `get_deployment_record(state_path, agent_name)`

Return a deployment record for a specific agent.

Source code in `src/holodeck/deploy/state.py`

```
def get_deployment_record(state_path: Path, agent_name: str) -> DeploymentRecord | None:
    """Return a deployment record for a specific agent."""
    state = load_state(state_path)
    return state.deployments.get(agent_name)
```

### update_deployment_record

## `update_deployment_record(state_path, agent_name, record)`

Update deployment record for an agent and persist it.

Source code in `src/holodeck/deploy/state.py`

```
def update_deployment_record(
    state_path: Path, agent_name: str, record: DeploymentRecord
) -> DeploymentRecord:
    """Update deployment record for an agent and persist it."""
    state = load_state(state_path)
    existing = state.deployments.get(agent_name)
    now = datetime.now(UTC)

    created_at = record.created_at or (existing.created_at if existing else None) or now
    updated_record = record.model_copy(
        update={"created_at": created_at, "updated_at": now}
    )

    state.deployments[agent_name] = updated_record
    if not state.version:
        state = state.model_copy(update={"version": STATE_VERSION})
    save_state(state_path, state)
    return updated_record
```

______________________________________________________________________

## `holodeck.deploy.deployers` -- Cloud deployers

Factory module that instantiates the correct deployer based on cloud provider.

### create_deployer

## `create_deployer(target)`

Create a cloud deployer based on the target configuration.

Source code in `src/holodeck/deploy/deployers/__init__.py`

```
def create_deployer(target: CloudTargetConfig) -> BaseDeployer:
    """Create a cloud deployer based on the target configuration."""
    if target.provider == CloudProvider.AZURE:
        if not target.azure:
            raise DeploymentError(
                operation="deploy",
                message="Azure configuration is required for Azure deployments.",
            )
        from holodeck.deploy.deployers.azure_containerapps import (
            AzureContainerAppsDeployer,
        )

        return AzureContainerAppsDeployer(target.azure)

    if target.provider == CloudProvider.AWS:
        raise DeploymentError(
            operation="deploy",
            message=(
                "AWS App Runner deployer is not implemented yet. "
                "Azure Container Apps is the only supported provider for now."
            ),
        )

    if target.provider == CloudProvider.GCP:
        raise DeploymentError(
            operation="deploy",
            message=(
                "GCP Cloud Run deployer is not implemented yet. "
                "Azure Container Apps is the only supported provider for now."
            ),
        )

    raise DeploymentError(
        operation="deploy",
        message=f"Unsupported cloud provider: {target.provider}",
    )
```

______________________________________________________________________

## `holodeck.deploy.deployers.base` -- Base deployer interface

### BaseDeployer

## `BaseDeployer`

Bases: `ABC`

Abstract base class for cloud deployers.

### `deploy(*, service_name, image_uri, port, env_vars, health_check_path='/health', **kwargs)`

Deploy a containerized service and return deployment details.

Parameters:

| Name                | Type             | Description                                     | Default     |
| ------------------- | ---------------- | ----------------------------------------------- | ----------- |
| `service_name`      | `str`            | Name for the deployed service.                  | *required*  |
| `image_uri`         | `str`            | Full container image URI including tag.         | *required*  |
| `port`              | `int`            | Container port to expose.                       | *required*  |
| `env_vars`          | `dict[str, str]` | Environment variables to set in the container.  | *required*  |
| `health_check_path` | `str`            | HTTP path for health checks (default: /health). | `'/health'` |
| `**kwargs`          | `Any`            | Provider-specific deployment options.           | `{}`        |

Returns:

| Type           | Description                                                        |
| -------------- | ------------------------------------------------------------------ |
| `DeployResult` | DeployResult containing service_id, service_name, url, and status. |

Raises:

| Type              | Description          |
| ----------------- | -------------------- |
| `DeploymentError` | If deployment fails. |

Source code in `src/holodeck/deploy/deployers/base.py`

```
@abstractmethod
def deploy(
    self,
    *,
    service_name: str,
    image_uri: str,
    port: int,
    env_vars: dict[str, str],
    health_check_path: str = "/health",
    **kwargs: Any,
) -> DeployResult:
    """Deploy a containerized service and return deployment details.

    Args:
        service_name: Name for the deployed service.
        image_uri: Full container image URI including tag.
        port: Container port to expose.
        env_vars: Environment variables to set in the container.
        health_check_path: HTTP path for health checks (default: /health).
        **kwargs: Provider-specific deployment options.

    Returns:
        DeployResult containing service_id, service_name, url, and status.

    Raises:
        DeploymentError: If deployment fails.
    """
```

### `get_status(service_id)`

Retrieve deployment status and URL by service identifier.

Parameters:

| Name         | Type  | Description                                 | Default    |
| ------------ | ----- | ------------------------------------------- | ---------- |
| `service_id` | `str` | Unique identifier for the deployed service. | *required* |

Returns:

| Type           | Description                                     |
| -------------- | ----------------------------------------------- |
| `StatusResult` | StatusResult containing current status and URL. |

Raises:

| Type              | Description            |
| ----------------- | ---------------------- |
| `DeploymentError` | If status check fails. |

Source code in `src/holodeck/deploy/deployers/base.py`

```
@abstractmethod
def get_status(self, service_id: str) -> StatusResult:
    """Retrieve deployment status and URL by service identifier.

    Args:
        service_id: Unique identifier for the deployed service.

    Returns:
        StatusResult containing current status and URL.

    Raises:
        DeploymentError: If status check fails.
    """
```

### `destroy(service_id)`

Destroy a deployed service by identifier.

Parameters:

| Name         | Type  | Description                                 | Default    |
| ------------ | ----- | ------------------------------------------- | ---------- |
| `service_id` | `str` | Unique identifier for the deployed service. | *required* |

Raises:

| Type              | Description                 |
| ----------------- | --------------------------- |
| `DeploymentError` | If destroy operation fails. |

Source code in `src/holodeck/deploy/deployers/base.py`

```
@abstractmethod
def destroy(self, service_id: str) -> None:
    """Destroy a deployed service by identifier.

    Args:
        service_id: Unique identifier for the deployed service.

    Raises:
        DeploymentError: If destroy operation fails.
    """
```

### `stream_logs(service_id)`

Stream deployment logs for a service identifier.

Parameters:

| Name         | Type  | Description                                 | Default    |
| ------------ | ----- | ------------------------------------------- | ---------- |
| `service_id` | `str` | Unique identifier for the deployed service. | *required* |

Returns:

| Type            | Description            |
| --------------- | ---------------------- |
| `Iterable[str]` | Iterable of log lines. |

Raises:

| Type                  | Description                                          |
| --------------------- | ---------------------------------------------------- |
| `NotImplementedError` | When log streaming is not supported by the provider. |
| `DeploymentError`     | If log streaming fails.                              |

Source code in `src/holodeck/deploy/deployers/base.py`

```
@abstractmethod
def stream_logs(self, service_id: str) -> Iterable[str]:
    """Stream deployment logs for a service identifier.

    Args:
        service_id: Unique identifier for the deployed service.

    Returns:
        Iterable of log lines.

    Raises:
        NotImplementedError: When log streaming is not supported by the provider.
        DeploymentError: If log streaming fails.
    """
```

______________________________________________________________________

## `holodeck.deploy.deployers.azure_containerapps` -- Azure Container Apps

### AzureContainerAppsDeployer

## `AzureContainerAppsDeployer(config)`

Bases: `BaseDeployer`

Deploy HoloDeck agents to Azure Container Apps.

Initialize Azure Container Apps deployer.

Parameters:

| Name     | Type                       | Description                        | Default    |
| -------- | -------------------------- | ---------------------------------- | ---------- |
| `config` | `AzureContainerAppsConfig` | Azure Container Apps configuration | *required* |

Raises:

| Type                        | Description                                 |
| --------------------------- | ------------------------------------------- |
| `DeploymentError`           | If configuration is missing required values |
| `CloudSDKNotInstalledError` | If Azure SDK dependencies are missing       |

Source code in `src/holodeck/deploy/deployers/azure_containerapps.py`

```
def __init__(self, config: AzureContainerAppsConfig) -> None:
    """Initialize Azure Container Apps deployer.

    Args:
        config: Azure Container Apps configuration

    Raises:
        DeploymentError: If configuration is missing required values
        CloudSDKNotInstalledError: If Azure SDK dependencies are missing
    """
    if not config.environment_name:
        raise DeploymentError(
            operation="deploy",
            message="Azure Container Apps requires environment_name to be set.",
        )

    try:
        from azure.core.exceptions import (
            ClientAuthenticationError,
            HttpResponseError,
            ResourceNotFoundError,
            ServiceRequestError,
        )
        from azure.identity import DefaultAzureCredential
        from azure.mgmt.appcontainers import ContainerAppsAPIClient
        from azure.mgmt.appcontainers.models import (
            Configuration,
            Container,
            ContainerApp,
            ContainerAppProbe,
            ContainerAppProbeHttpGet,
            ContainerAppProperties,
            ContainerResources,
            EnvironmentVar,
            Ingress,
            Scale,
            StorageType,
            Template,
            TrafficWeight,
            Volume,
            VolumeMount,
        )
    except ImportError as exc:
        raise CloudSDKNotInstalledError(
            provider="azure", sdk_name="azure-mgmt-appcontainers"
        ) from exc

    # Store exception types for use in methods
    self._ClientAuthenticationError: type[ClientAuthenticationError] = (
        ClientAuthenticationError
    )
    self._HttpResponseError: type[HttpResponseError] = HttpResponseError
    self._ResourceNotFoundError: type[ResourceNotFoundError] = ResourceNotFoundError
    self._ServiceRequestError: type[ServiceRequestError] = ServiceRequestError

    self._config = config
    self._client: ContainerAppsAPIClient = ContainerAppsAPIClient(
        DefaultAzureCredential(), config.subscription_id
    )
    self._Configuration: type[Configuration] = Configuration
    self._Container: type[Container] = Container
    self._ContainerApp: type[ContainerApp] = ContainerApp
    self._ContainerAppProperties: type[ContainerAppProperties] = (
        ContainerAppProperties
    )
    self._ContainerAppProbe: type[ContainerAppProbe] = ContainerAppProbe
    self._ContainerAppProbeHttpGet: type[ContainerAppProbeHttpGet] = (
        ContainerAppProbeHttpGet
    )
    self._ContainerResources: type[ContainerResources] = ContainerResources
    self._EnvironmentVar: type[EnvironmentVar] = EnvironmentVar
    self._Ingress: type[Ingress] = Ingress
    self._Scale: type[Scale] = Scale
    self._StorageType: type[StorageType] = StorageType
    self._Template: type[Template] = Template
    self._TrafficWeight: type[TrafficWeight] = TrafficWeight
    self._Volume: type[Volume] = Volume
    self._VolumeMount: type[VolumeMount] = VolumeMount
    self._environment_id = (
        f"/subscriptions/{config.subscription_id}/resourceGroups/"
        f"{config.resource_group}/providers/Microsoft.App/managedEnvironments/"
        f"{config.environment_name}"
    )
```

### `deploy(*, service_name, image_uri, port, env_vars, health_check_path='/health', readiness_path='/ready', **kwargs)`

Deploy a container to Azure Container Apps.

Source code in `src/holodeck/deploy/deployers/azure_containerapps.py`

```
def deploy(
    self,
    *,
    service_name: str,
    image_uri: str,
    port: int,
    env_vars: dict[str, str],
    health_check_path: str = "/health",
    readiness_path: str = "/ready",
    **kwargs: Any,
) -> DeployResult:
    """Deploy a container to Azure Container Apps."""
    self._echo_resolved_config(
        service_name=service_name, image_uri=image_uri, port=port
    )

    env_list = [
        self._EnvironmentVar(name=key, value=value)
        for key, value in env_vars.items()
    ]

    # Liveness probe — "is this process still alive". Failure → ACA
    # restarts the replica.
    liveness_probe = self._ContainerAppProbe(
        type="Liveness",
        http_get=self._ContainerAppProbeHttpGet(
            port=port,
            path=health_check_path,
        ),
        initial_delay_seconds=10,
        period_seconds=30,
        failure_threshold=3,
        timeout_seconds=5,
    )

    # Readiness probe — "is this replica ready to receive traffic". The
    # serve layer's /ready endpoint reflects server state; ACA holds
    # traffic off until this returns 200 so cold replicas don't race
    # their first request against init work (spec 034 P1a).
    readiness_probe = self._ContainerAppProbe(
        type="Readiness",
        http_get=self._ContainerAppProbeHttpGet(
            port=port,
            path=readiness_path,
        ),
        initial_delay_seconds=5,
        period_seconds=10,
        failure_threshold=3,
        timeout_seconds=5,
    )

    # Spec 034 P2a — ephemeral writable scratch (EmptyDir). ACA does not
    # expose tmpfs directly, but EmptyDir is per-replica and cleared on
    # replica restart, which gives the same operational property: tool
    # outputs and SDK scratch never persist across replicas. See the
    # research note in spec 034 P2 plan Task 11 for why this is the only
    # securityContext-adjacent primitive ACA exposes.
    volumes = [
        self._Volume(name="tmp", storage_type=self._StorageType.EMPTY_DIR),
        self._Volume(name="sdk-scratch", storage_type=self._StorageType.EMPTY_DIR),
    ]
    volume_mounts = [
        self._VolumeMount(
            volume_name="tmp", mount_path="/tmp"  # noqa: S108  # nosec B108
        ),
        self._VolumeMount(
            volume_name="sdk-scratch", mount_path="/var/holodeck/work"
        ),
    ]

    container = self._Container(
        name=service_name,
        image=image_uri,
        resources=self._ContainerResources(
            cpu=self._config.cpu,
            memory=self._config.memory,
        ),
        env=env_list if env_list else None,
        probes=[liveness_probe, readiness_probe],
        volume_mounts=volume_mounts,
    )

    scale = self._Scale(
        min_replicas=self._config.min_replicas,
        max_replicas=self._config.max_replicas,
    )

    template = self._Template(containers=[container], scale=scale, volumes=volumes)
    ingress = self._Ingress(
        external=self._config.ingress_external,
        target_port=port,
        traffic=[self._TrafficWeight(weight=100, latest_revision=True)],
        # azure-mgmt-appcontainers 5 stopped sending these defaults; send
        # them explicitly so the request matches what 4.x sent.
        allow_insecure=False,
        transport="auto",
    )
    configuration = self._Configuration(
        ingress=ingress, active_revisions_mode="Single"
    )

    # SDK 5 types these fields under ``properties``; passing them flat
    # only works through its runtime compatibility shim.
    container_app = self._ContainerApp(
        location=self._config.location,
        properties=self._ContainerAppProperties(
            managed_environment_id=self._environment_id,
            configuration=configuration,
            template=template,
        ),
    )

    try:
        poller = self._client.container_apps.begin_create_or_update(
            resource_group_name=self._config.resource_group,
            container_app_name=service_name,
            container_app_envelope=container_app,
        )
        result = poller.result()
    except self._ClientAuthenticationError as exc:
        raise DeploymentError(
            operation="deploy",
            message=f"Azure authentication failed: {exc}. "
            "Check your Azure credentials configuration.",
        ) from exc
    except self._ServiceRequestError as exc:
        raise DeploymentError(
            operation="deploy",
            message=f"Network error connecting to Azure: {exc}. "
            "Check your network connectivity.",
        ) from exc
    except self._HttpResponseError as exc:
        raise DeploymentError(
            operation="deploy",
            message=f"Azure API error (HTTP {exc.status_code}): {exc.message}",
        ) from exc
    except Exception as exc:
        raise DeploymentError(
            operation="deploy",
            message=f"Azure Container Apps deployment failed: {exc}",
        ) from exc

    url = None
    if result.configuration and result.configuration.ingress:
        fqdn = result.configuration.ingress.fqdn
        if fqdn:
            url = f"https://{fqdn}"

    result_service_id = result.id or result.name or service_name
    result_service_name = result.name or service_name
    result_status = _provisioning_status(result.provisioning_state)

    return DeployResult(
        service_id=result_service_id,
        service_name=result_service_name,
        url=url,
        status=result_status,
    )
```

### `get_status(service_id)`

Get status for an Azure Container App.

Source code in `src/holodeck/deploy/deployers/azure_containerapps.py`

```
def get_status(self, service_id: str) -> StatusResult:
    """Get status for an Azure Container App."""
    container_app_name = self._resolve_container_app_name(service_id)
    try:
        app = self._client.container_apps.get(
            resource_group_name=self._config.resource_group,
            container_app_name=container_app_name,
        )
    except self._ResourceNotFoundError as exc:
        raise DeploymentError(
            operation="status",
            message=f"Container app '{container_app_name}' not found: {exc}",
        ) from exc
    except self._ClientAuthenticationError as exc:
        raise DeploymentError(
            operation="status",
            message=f"Azure authentication failed: {exc}. "
            "Check your Azure credentials configuration.",
        ) from exc
    except self._ServiceRequestError as exc:
        raise DeploymentError(
            operation="status",
            message=f"Network error connecting to Azure: {exc}. "
            "Check your network connectivity.",
        ) from exc
    except self._HttpResponseError as exc:
        raise DeploymentError(
            operation="status",
            message=f"Azure API error (HTTP {exc.status_code}): {exc.message}",
        ) from exc
    except Exception as exc:
        raise DeploymentError(
            operation="status",
            message=f"Failed to fetch Azure deployment status: {exc}",
        ) from exc

    url = None
    if app.configuration and app.configuration.ingress:
        fqdn = app.configuration.ingress.fqdn
        if fqdn:
            url = f"https://{fqdn}"

    status = _provisioning_status(app.provisioning_state)
    return StatusResult(status=status, url=url)
```

### `destroy(service_id)`

Destroy an Azure Container App deployment.

Source code in `src/holodeck/deploy/deployers/azure_containerapps.py`

```
def destroy(self, service_id: str) -> None:
    """Destroy an Azure Container App deployment."""
    container_app_name = self._resolve_container_app_name(service_id)
    try:
        poller = self._client.container_apps.begin_delete(
            resource_group_name=self._config.resource_group,
            container_app_name=container_app_name,
        )
        poller.result()
    except self._ResourceNotFoundError as exc:
        raise DeploymentError(
            operation="destroy",
            message=f"Container app '{container_app_name}' not found: {exc}",
        ) from exc
    except self._ClientAuthenticationError as exc:
        raise DeploymentError(
            operation="destroy",
            message=f"Azure authentication failed: {exc}. "
            "Check your Azure credentials configuration.",
        ) from exc
    except self._ServiceRequestError as exc:
        raise DeploymentError(
            operation="destroy",
            message=f"Network error connecting to Azure: {exc}. "
            "Check your network connectivity.",
        ) from exc
    except self._HttpResponseError as exc:
        raise DeploymentError(
            operation="destroy",
            message=f"Azure API error (HTTP {exc.status_code}): {exc.message}",
        ) from exc
    except Exception as exc:
        raise DeploymentError(
            operation="destroy",
            message=f"Failed to destroy Azure deployment: {exc}",
        ) from exc
```

### `stream_logs(service_id)`

Stream logs for Azure Container Apps (not implemented).

Source code in `src/holodeck/deploy/deployers/azure_containerapps.py`

```
def stream_logs(self, service_id: str) -> Iterable[str]:
    """Stream logs for Azure Container Apps (not implemented)."""
    raise NotImplementedError(
        "Log streaming is not implemented for Azure Container Apps."
    )
```

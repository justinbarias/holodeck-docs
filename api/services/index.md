# Services

The `holodeck.services` package contains client classes for interacting with external services such as the MCP Registry API.

## MCP Registry Client

`MCPRegistryClient` provides methods to search, retrieve, and list MCP servers from the official registry at <https://registry.modelcontextprotocol.io>.

## `MCPRegistryClient(base_url=DEFAULT_BASE_URL, timeout=DEFAULT_TIMEOUT)`

Client for MCP Registry API.

Provides methods to search, retrieve, and list MCP servers from the official registry.

Example

> > > client = MCPRegistryClient() result = client.search(query="filesystem") for server in result.servers: ... print(f"{server.name}: {server.description}")

Initialize client with base URL and timeout.

Parameters:

| Name       | Type    | Description                               | Default            |
| ---------- | ------- | ----------------------------------------- | ------------------ |
| `base_url` | `str`   | Registry API base URL                     | `DEFAULT_BASE_URL` |
| `timeout`  | `float` | Request timeout in seconds (default: 5.0) | `DEFAULT_TIMEOUT`  |

Source code in `src/holodeck/services/mcp_registry.py`

```
def __init__(
    self,
    base_url: str = DEFAULT_BASE_URL,
    timeout: float = DEFAULT_TIMEOUT,
) -> None:
    """Initialize client with base URL and timeout.

    Args:
        base_url: Registry API base URL
        timeout: Request timeout in seconds (default: 5.0)
    """
    self.base_url = base_url.rstrip("/")
    self.timeout = timeout
    self._session = requests.Session()
```

### `search(query=None, limit=25, cursor=None)`

Search for MCP servers.

Parameters:

| Name     | Type  | Description                           | Default                                        |
| -------- | ----- | ------------------------------------- | ---------------------------------------------- |
| `query`  | \`str | None\`                                | Optional search term (substring match on name) |
| `limit`  | `int` | Maximum results per page (default 25) | `25`                                           |
| `cursor` | \`str | None\`                                | Pagination cursor for next page                |

Returns:

| Type           | Description                                   |
| -------------- | --------------------------------------------- |
| `SearchResult` | SearchResult with servers and pagination info |

Raises:

| Type                      | Description               |
| ------------------------- | ------------------------- |
| `RegistryConnectionError` | Network/timeout issues    |
| `RegistryAPIError`        | API returned error status |

Source code in `src/holodeck/services/mcp_registry.py`

```
def search(
    self,
    query: str | None = None,
    limit: int = 25,
    cursor: str | None = None,
) -> SearchResult:
    """Search for MCP servers.

    Args:
        query: Optional search term (substring match on name)
        limit: Maximum results per page (default 25)
        cursor: Pagination cursor for next page

    Returns:
        SearchResult with servers and pagination info

    Raises:
        RegistryConnectionError: Network/timeout issues
        RegistryAPIError: API returned error status
    """
    logger.debug("Searching MCP registry: query=%s, limit=%d", query, limit)
    url = f"{self.base_url}/v0.1/servers"
    params: dict[str, str | int] = {"limit": limit}

    if query:
        params["search"] = query
    if cursor:
        params["cursor"] = cursor

    response = self._request("GET", url, params=params)
    data = response.json()

    # Parse response structure
    servers: list[RegistryServer] = []
    for item in data.get("servers", []):
        server_data = item.get("server", item)
        meta_data = item.get("_meta")
        servers.append(self._parse_server(server_data, meta_data))

    # Aggregate servers by name (each version comes as separate entry)
    servers = self._aggregate_by_name(servers)

    metadata = data.get("metadata", {})
    result = SearchResult(
        servers=servers,
        next_cursor=metadata.get("nextCursor"),
        total_count=metadata.get("count", len(servers)),
    )
    logger.debug("Search returned %d unique servers", len(result.servers))
    return result
```

### `get_server(name, version='latest')`

Get specific server by name and version.

Parameters:

| Name      | Type  | Description                      | Default    |
| --------- | ----- | -------------------------------- | ---------- |
| `name`    | `str` | Server name (reverse-DNS format) | *required* |
| `version` | `str` | Version string or "latest"       | `'latest'` |

Returns:

| Type             | Description                      |
| ---------------- | -------------------------------- |
| `RegistryServer` | RegistryServer with full details |

Raises:

| Type                      | Description            |
| ------------------------- | ---------------------- |
| `ServerNotFoundError`     | Server doesn't exist   |
| `RegistryConnectionError` | Network/timeout issues |

Source code in `src/holodeck/services/mcp_registry.py`

```
def get_server(
    self,
    name: str,
    version: str = "latest",
) -> RegistryServer:
    """Get specific server by name and version.

    Args:
        name: Server name (reverse-DNS format)
        version: Version string or "latest"

    Returns:
        RegistryServer with full details

    Raises:
        ServerNotFoundError: Server doesn't exist
        RegistryConnectionError: Network/timeout issues
    """
    logger.debug("Fetching server: name=%s, version=%s", name, version)
    # URL-encode the server name (contains '/' in reverse-DNS format)
    encoded_name = quote(name, safe="")
    url = f"{self.base_url}/v0.1/servers/{encoded_name}/versions/{version}"

    try:
        response = self._request("GET", url)
    except RegistryAPIError as e:
        if e.status_code == 404:
            logger.debug("Server not found: %s", name)
            raise ServerNotFoundError(name) from e
        raise

    data = response.json()
    server_data = data.get("server", data)
    server = self._parse_server(server_data, data.get("_meta"))
    logger.debug("Retrieved server: %s v%s", server.name, server.version)
    return server
```

### `list_versions(name)`

List available versions for a server.

Parameters:

| Name   | Type  | Description                      | Default    |
| ------ | ----- | -------------------------------- | ---------- |
| `name` | `str` | Server name (reverse-DNS format) | *required* |

Returns:

| Type        | Description                            |
| ----------- | -------------------------------------- |
| `list[str]` | List of version strings (newest first) |

Raises:

| Type                      | Description            |
| ------------------------- | ---------------------- |
| `ServerNotFoundError`     | Server doesn't exist   |
| `RegistryConnectionError` | Network/timeout issues |

Source code in `src/holodeck/services/mcp_registry.py`

```
def list_versions(self, name: str) -> list[str]:
    """List available versions for a server.

    Args:
        name: Server name (reverse-DNS format)

    Returns:
        List of version strings (newest first)

    Raises:
        ServerNotFoundError: Server doesn't exist
        RegistryConnectionError: Network/timeout issues
    """
    logger.debug("Listing versions for server: %s", name)
    # URL-encode the server name (contains '/' in reverse-DNS format)
    encoded_name = quote(name, safe="")
    url = f"{self.base_url}/v0.1/servers/{encoded_name}/versions"

    try:
        response = self._request("GET", url)
    except RegistryAPIError as e:
        if e.status_code == 404:
            logger.debug("Server not found: %s", name)
            raise ServerNotFoundError(name) from e
        raise

    data = response.json()
    versions: list[str] = []
    # API returns servers array with version info embedded in each server object
    for item in data.get("servers", []):
        server_data = item.get("server", item)
        version_str = server_data.get("version")
        if version_str:
            versions.append(version_str)
    logger.debug("Found %d versions for %s", len(versions), name)
    return versions
```

### `close()`

Close the underlying HTTP session.

Should be called when done with the client to release resources. Alternatively, use the client as a context manager.

Example

> > > with MCPRegistryClient() as client: ... result = client.search("filesystem")

#### Session automatically closed

Source code in `src/holodeck/services/mcp_registry.py`

```
def close(self) -> None:
    """Close the underlying HTTP session.

    Should be called when done with the client to release resources.
    Alternatively, use the client as a context manager.

    Example:
        >>> with MCPRegistryClient() as client:
        ...     result = client.search("filesystem")
        # Session automatically closed
    """
    if hasattr(self, "_session") and self._session is not None:
        self._session.close()
```

## Module-Level Functions

### find_stdio_package

## `find_stdio_package(server)`

Find a package that supports stdio transport.

Prioritizes supported registry types (npm, pypi, docker, oci) when multiple stdio packages are available.

Parameters:

| Name     | Type             | Description                   | Default    |
| -------- | ---------------- | ----------------------------- | ---------- |
| `server` | `RegistryServer` | Registry server with packages | *required* |

Returns:

| Type                    | Description |
| ----------------------- | ----------- |
| \`RegistryServerPackage | None\`      |
| \`RegistryServerPackage | None\`      |

Source code in `src/holodeck/services/mcp_registry.py`

```
def find_stdio_package(server: RegistryServer) -> RegistryServerPackage | None:
    """Find a package that supports stdio transport.

    Prioritizes supported registry types (npm, pypi, docker, oci) when
    multiple stdio packages are available.

    Args:
        server: Registry server with packages

    Returns:
        First stdio-compatible package with supported registry type,
        or any stdio package as fallback, or None if none found
    """
    # First pass: find stdio package with supported registry type
    for pkg in server.packages:
        if (
            pkg.transport.type == "stdio"
            and pkg.registry_type in SUPPORTED_REGISTRY_TYPES
        ):
            return pkg

    # Fallback: any stdio package (will fail later with unsupported type error)
    for pkg in server.packages:
        if pkg.transport.type == "stdio":
            return pkg

    return None
```

### registry_to_mcp_tool

## `registry_to_mcp_tool(server, package=None, transport_override=None)`

Convert registry server to MCPTool configuration.

Transforms an MCP server from the registry into a tool configuration that can be added to agent.yaml or global config.

Parameters:

| Name                 | Type                    | Description                      | Default                                             |
| -------------------- | ----------------------- | -------------------------------- | --------------------------------------------------- |
| `server`             | `RegistryServer`        | RegistryServer from registry API | *required*                                          |
| `package`            | \`RegistryServerPackage | None\`                           | Specific package to use (defaults to first package) |
| `transport_override` | \`str                   | None\`                           | Optional transport type override                    |

Returns:

| Type      | Description                                        |
| --------- | -------------------------------------------------- |
| `MCPTool` | MCPTool configuration ready for YAML serialization |

Raises:

| Type         | Description               |
| ------------ | ------------------------- |
| `ValueError` | If server has no packages |

Source code in `src/holodeck/services/mcp_registry.py`

```
def registry_to_mcp_tool(
    server: RegistryServer,
    package: RegistryServerPackage | None = None,
    transport_override: str | None = None,
) -> MCPTool:
    """Convert registry server to MCPTool configuration.

    Transforms an MCP server from the registry into a tool configuration
    that can be added to agent.yaml or global config.

    Args:
        server: RegistryServer from registry API
        package: Specific package to use (defaults to first package)
        transport_override: Optional transport type override

    Returns:
        MCPTool configuration ready for YAML serialization

    Raises:
        ValueError: If server has no packages
    """
    if not server.packages:
        raise ValueError(f"Server '{server.name}' has no packages configured")

    # Use specified package or default to first
    pkg = package or server.packages[0]

    # Map transport type (with override support)
    transport_map: dict[str, TransportType] = {
        "stdio": TransportType.STDIO,
        "sse": TransportType.SSE,
        "streamable-http": TransportType.HTTP,
    }

    if transport_override:
        transport = transport_map.get(transport_override, TransportType.STDIO)
    else:
        transport = transport_map.get(pkg.transport.type, TransportType.STDIO)

    # Map registry type to command
    # OCI is treated like docker (container images)
    command_map: dict[str, CommandType] = {
        "npm": CommandType.NPX,
        "pypi": CommandType.UVX,
        "docker": CommandType.DOCKER,
        "oci": CommandType.DOCKER,
    }
    command = command_map.get(pkg.registry_type)

    # Validate that we have a supported registry type for stdio transport
    if transport == TransportType.STDIO and command is None:
        raise ValueError(
            f"Unsupported registry type '{pkg.registry_type}' for stdio transport. "
            f"Supported types: {', '.join(sorted(SUPPORTED_REGISTRY_TYPES))}"
        )

    # Build args based on registry type
    args: list[str] = []
    if pkg.registry_type == "npm":
        version_suffix = f"@{pkg.version}" if pkg.version else "@latest"
        args = ["-y", f"{pkg.identifier}{version_suffix}"]
    elif pkg.registry_type == "pypi":
        args = [f"{pkg.identifier}=={pkg.version}"] if pkg.version else [pkg.identifier]
    elif pkg.registry_type in ("docker", "oci"):
        # OCI identifiers may already include the tag
        if ":" in pkg.identifier:
            args = ["run", "-i", pkg.identifier]
        else:
            version_tag = pkg.version or "latest"
            args = ["run", "-i", f"{pkg.identifier}:{version_tag}"]

    # Extract env vars as placeholders
    env: dict[str, str] | None = None
    if pkg.environment_variables:
        env = {ev.name: f"${{{ev.name}}}" for ev in pkg.environment_variables}

    # Validate server name before extracting short name
    if not server.name:
        raise ValueError("Server name is required")

    # Extract short name from reverse-DNS format (e.g., "io.github.user/server-name")
    raw_name = server.name.split("/")[-1] if "/" in server.name else server.name

    # Sanitize to valid tool name (alphanumeric and underscores only)
    short_name = sanitize_tool_name(raw_name)

    return MCPTool(
        name=short_name,
        description=server.description,
        type="mcp",
        transport=transport,
        command=command if transport == TransportType.STDIO else None,
        args=args if transport == TransportType.STDIO else None,
        env=env,
        env_file=None,
        encoding=None,
        url=pkg.transport.url if transport != TransportType.STDIO else None,
        headers=None,
        timeout=None,
        sse_read_timeout=None,
        terminate_on_close=None,
        config=None,
        load_tools=True,
        load_prompts=True,
        request_timeout=60,
        is_retrieval=False,
        registry_name=server.name,  # Store full name for duplicate detection
    )
```

## Constants

### SUPPORTED_REGISTRY_TYPES

## `SUPPORTED_REGISTRY_TYPES = frozenset({'npm', 'pypi', 'docker', 'oci'})`

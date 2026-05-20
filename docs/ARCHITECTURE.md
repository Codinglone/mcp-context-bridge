# Context Bridge — Architecture

> **Version:** 0.1.0  
> **Status:** Beta — all 6 connectors implemented, HTTP transport live, browser extension scaffolded

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MCP Client                                    │
│         (Claude Desktop / Cursor / Aether / Web Chatbots)             │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   │ MCP Protocol (stdio / HTTP)
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    Context Bridge Server                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
│  │   MCP       │  │   Router    │  │   Cache     │  │   Config  │ │
│  │   Transport │──│   /         │──│   Layer     │  │   Manager │ │
│  │   Layer     │  │   Dispatcher│  │   (TTL)     │  │   (YAML)  │ │
│  └─────────────┘  └──────┬──────┘  └─────────────┘  └───────────┘ │
│                          │                                         │
│           ┌──────────────┼──────────────┐                         │
│           ▼              ▼              ▼                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │ Filesystem  │  │   GitHub    │  │    SSH      │               │
│  │ Connector   │  │  Connector  │  │  Connector  │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
│                                                                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐               │
│  │  Obsidian   │  │  PostgreSQL │  │    Docker   │               │
│  │  Connector  │  │  Connector  │  │  Connector  │               │
│  └─────────────┘  └─────────────┘  └─────────────┘               │
└─────────────────────────────────────────────────────────────────────┘
         ▲
         │ HTTP (localhost:8080)
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│              Browser Extension (Chrome/Firefox)                    │
│  Injects @context trigger into ChatGPT / Claude / Poe web UIs     │
└─────────────────────────────────────────────────────────────────────┘
```

## Components

### 1. Transport Layer

Handles MCP protocol communication.

- **stdio**: For CLI-based clients (Claude Desktop, local scripts)
- **HTTP**: For web-based clients and browser extensions
- **Request/Response**: Parse MCP `tools/list`, `tools/call` methods

**Library:** `mcp` (official Python SDK) for stdio; Starlette + Uvicorn for HTTP.

### 2. Router / Dispatcher

Central request handler.

- Receives MCP tool calls (e.g., `fs.read_file`, `github.get_pr`)
- Dispatches to the appropriate connector
- Applies caching
- Formats responses as MCP-compliant JSON

### 3. Cache Layer

Prevents redundant expensive operations.

- **In-memory dict** with TTL (default 60s for most operations)
- **File contents**: Cache by path + mtime hash
- **GitHub API**: Cache by URL + etag
- **Obsidian index**: Rebuild on vault file changes only
- **PostgreSQL schema**: Cache for 5 minutes
- **Docker containers**: Cache for 10 seconds

### 4. Configuration Manager

Loads and validates user configuration.

- **Source:** `~/.config/context-bridge/config.yaml`
- **Validation:** Pydantic settings with typed models
- **Env var interpolation:** `${GITHUB_TOKEN}` etc.

### 5. Connectors

All connectors inherit from `BaseConnector` and implement:
- `initialize()` / `shutdown()` — lifecycle
- `get_tools()` — return MCP tool definitions
- `call_tool()` — execute tool calls

| Connector | Library | Key Feature |
|-----------|---------|-------------|
| **Filesystem** | `watchdog` | Path traversal prevention, binary detection |
| **GitHub** | `httpx` | Base64 decoding, pagination, code search |
| **SSH** | `paramiko` | Connection pooling with keepalive |
| **Obsidian** | `frontmatter` + `watchdog` | Wiki-link graph, full-text index |
| **PostgreSQL** | `psycopg` | Schema introspection, read-only safety, SSH tunnels |
| **Docker** | `docker` | Compose service detection |

## Data Flow

### Tool Call Flow (stdio)

```
1. Client sends: tools/call { "name": "fs.read_file", "arguments": { "path": "src/main.py" } }
2. Transport layer parses JSON-RPC message
3. Router looks up "fs.read_file" → FilesystemConnector
4. Cache check: do we have src/main.py cached and fresh?
   - Yes → return cached content
   - No → read from disk, store in cache, return
5. Router formats response as MCP tool result
6. Transport sends back to client
```

### Tool Call Flow (HTTP → Browser Extension)

```
1. User clicks @context button on ChatGPT page
2. Extension opens modal, user selects "fs.read_file"
3. Extension POSTs to localhost:8080/mcp/v1/tools/fs.read_file
4. Router dispatches → FilesystemConnector
5. Result returned as JSON
6. Extension inserts result into chat input
```

### File Watch Flow

```
1. watchdog detects change in ~/projects/my-app/src/main.py
2. Connector invalidates cache entry for that path
3. Next tool call reads fresh data
```

## Technology Stack

| Component | Technology | Reason |
|-----------|-----------|--------|
| Language | Python 3.11+ | Fast iteration, great async support |
| Async | `asyncio` | Handle multiple concurrent connections |
| HTTP | `httpx` | Async HTTP client for GitHub/API calls |
| SSH | `paramiko` | Mature, pure-Python SSH |
| File Watching | `watchdog` | Cross-platform, reliable |
| Config | `pydantic-settings` + YAML | Validation + typed config |
| Cache | `cachetools` TTLCache | Simple, sufficient |
| MCP | `mcp` (official SDK) | Standard compliance |
| CLI | `typer` | Fast CLI building |
| HTTP Server | `starlette` + `uvicorn` | ASGI for browser extension |
| Testing | `pytest` + `pytest-asyncio` | Standard |
| Linting | `ruff` | Fast, replaces flake8/black/isort |
| PostgreSQL | `psycopg` | Async-capable, modern |
| Docker | `docker` (official SDK) | Full API coverage |

## Project Structure

```
context-bridge/
├── src/
│   └── context_bridge/
│       ├── __init__.py
│       ├── server.py              # MCP server entrypoint (stdio + HTTP)
│       ├── router.py              # Request dispatcher
│       ├── config.py              # Pydantic settings
│       ├── cache.py               # TTL cache wrapper
│       ├── cli.py                 # Typer CLI
│       ├── transport_http.py      # Starlette HTTP transport
│       └── connectors/
│           ├── __init__.py
│           ├── base.py             # Connector ABC
│           ├── filesystem.py
│           ├── github.py
│           ├── ssh.py
│           ├── obsidian.py
│           ├── postgresql.py       # Includes SSH tunnel support
│           └── docker.py
├── tests/
│   ├── test_connectors/
│   │   ├── test_base.py
│   │   ├── test_filesystem.py
│   │   ├── test_github.py
│   │   ├── test_obsidian.py
│   │   ├── test_obsidian_deep.py
│   │   ├── test_postgresql.py
│   │   ├── test_postgresql_tunnel.py
│   │   ├── test_ssh.py
│   │   ├── test_docker.py
│   │   ├── test_integration.py
│   │   ├── test_integration_postgres.py
│   │   └── test_integration_ssh.py
│   └── test_core/
│       ├── test_cache.py
│       ├── test_config.py
│       ├── test_router.py
│       └── test_http_transport.py
├── extensions/
│   ├── README.md
│   └── chrome/
│       ├── manifest.json
│       ├── background.js
│       ├── content.js
│       ├── content.css
│       ├── popup.html
│       ├── popup.js
│       ├── README.md
│       └── icons/
│           ├── icon16.png
│           ├── icon48.png
│           └── icon128.png
├── docs/
│   ├── DESIGN.md
│   └── ARCHITECTURE.md
├── .github/
│   └── workflows/
│       └── ci.yml
├── pyproject.toml
├── Makefile
├── README.md
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
└── .gitignore
```

## Security Considerations

1. **Local Only by Default**: HTTP server binds to localhost only
2. **No Credential Logging**: API tokens and SSH keys never logged
3. **Path Traversal Prevention**: All filesystem paths resolved and validated against configured roots
4. **Read-Only by Default**: No write operations in v1
5. **SSH Key Permissions**: Warn if keys are too permissive

## Performance Targets

| Metric | Target | Status |
|--------|--------|--------|
| Tool call latency (cached) | <10ms | ✅ Verified |
| Tool call latency (uncached filesystem) | <50ms | ✅ Verified |
| Tool call latency (GitHub API) | <2s | ✅ Verified |
| File watch → cache invalidation | <100ms | ✅ Verified |
| Startup time | <2s | ✅ Verified |
| Memory footprint (idle) | <100MB | ✅ Verified |
| Test suite | <10s | ✅ 104 tests in ~4s |

## Future Extensions

1. **Vector Search**: Semantic retrieval for Obsidian notes
2. **Write Operations**: Allow LLM to create GitHub issues, edit files (with approval)
3. **SQLite Connector**: Lightweight database introspection
4. **Browser Connector**: Read open browser tabs via CDP
5. **Kubernetes Connector**: Read pod logs, inspect deployments
6. **Firefox Extension**: Port Chrome extension to Firefox

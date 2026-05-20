# Context Bridge — Architecture

## Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        MCP Client                                    │
│              (Claude Desktop / Cursor / Aether)                      │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ MCP Protocol (stdio / HTTP / SSE)
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
│  ┌─────────────┐                                                  │
│  │  Obsidian   │                                                  │
│  │  Connector  │                                                  │
│  └─────────────┘                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## Components

### 1. Transport Layer

Handles MCP protocol communication.

- **stdio**: For CLI-based clients (Claude Desktop, local scripts)
- **HTTP/SSE**: For remote clients or browser-based interfaces
- **Request/Response**: Parse MCP `tools/list`, `tools/call`, `resources/read` methods

**Library:** `mcp` (official Python SDK) or custom implementation if SDK is too limiting.

### 2. Router / Dispatcher

Central request handler.

- Receives MCP tool calls (e.g., `filesystem.read_file`, `github.get_pr`)
- Authenticates the request (if using HTTP transport with auth)
- Dispatches to the appropriate connector
- Applies caching and rate limiting
- Formats responses as MCP-compliant JSON

### 3. Cache Layer

Prevents redundant expensive operations.

- **In-memory dict** with TTL (default 60s for most operations)
- **File contents**: Cache by path + mtime hash
- **GitHub API**: Cache by URL + etag
- **SSH commands**: Cache by (host, command, cwd) for short TTL (10s)
- **Obsidian index**: Rebuild on vault file changes only

### 4. Configuration Manager

Loads and validates user configuration.

- **Source:** `~/.config/context-bridge/config.yaml`
- **Contents:**
  ```yaml
  server:
    transport: stdio  # or http
    port: 8080
  
  connectors:
    filesystem:
      - path: ~/projects/my-app
        name: my-app
        exclude: ["node_modules", ".git"]
    
    github:
      token: ${GITHUB_TOKEN}
      repos: ["owner/repo"]
    
    ssh:
      - host: prod-server
        user: deploy
        key: ~/.ssh/id_rsa
    
    obsidian:
      vault: ~/Documents/Obsidian
      exclude: [".git", "attachments"]
  ```

### 5. Connectors

#### Filesystem Connector
- **Library:** `watchdog` for file system events
- **Capabilities:**
  - `fs.read_file(path)` — returns file contents
  - `fs.list_dir(path)` — returns tree structure
  - `fs.find(pattern)` — glob search
  - `fs.get_recent_changes(n)` — last n modified files
- **Constraints:** Respects `.gitignore`, max file size (1MB default), binary file detection

#### GitHub Connector
- **Library:** `PyGithub` or raw `httpx` + `cachecontrol`
- **Capabilities:**
  - `github.get_file(repo, path, ref)`
  - `github.list_issues(repo, state)`
  - `github.get_pr(repo, number)`
  - `github.search_code(query)`
- **Auth:** Personal access token or OAuth app
- **Rate limiting:** Respects GitHub's 5000 req/hour, caches aggressively

#### SSH Connector
- **Library:** `paramiko`
- **Capabilities:**
  - `ssh.run_command(host, command)`
  - `ssh.read_file(host, path)`
  - `ssh.list_dir(host, path)`
- **Connection pooling:** Reuse SSH connections per host (keepalive)
- **Security:** Keys only, no password storage. Agent forwarding optional.

#### Obsidian Connector
- **Library:** `markdown` + `frontmatter` + `watchdog`
- **Capabilities:**
  - `obsidian.search(query)` — full-text search
  - `obsidian.get_note(path)` — note content with metadata
  - `obsidian.get_backlinks(title)` — reverse link lookup
  - `obsidian.get_tags()` — list all tags
- **Index:** In-memory inverted index rebuilt on file changes
- **Vault parsing:** Respects Obsidian's `.obsidian/app.json` exclusions

## Data Flow

### Tool Call Flow

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

### File Watch Flow

```
1. watchdog detects change in ~/projects/my-app/src/main.py
2. FilesystemConnector invalidates cache entry for that path
3. (Optional) If HTTP transport active, push resource/update notification to subscribed clients
4. Next tool call for that file reads fresh data
```

## Technology Stack

| Component | Technology | Reason |
|-----------|-----------|--------|
| Language | Python 3.11+ | Fast iteration, great async support, rich ecosystem |
| Async | `asyncio` + `anyio` | Handle multiple concurrent connections |
| HTTP | `httpx` | Async HTTP client for GitHub/API calls |
| SSH | `paramiko` | Mature, pure-Python SSH |
| File Watching | `watchdog` | Cross-platform, reliable |
| Config | `pydantic-settings` + YAML | Validation + typed config |
| Cache | `cachetools` TTLCache | Simple, sufficient |
| MCP | `mcp` (official SDK) | Standard compliance |
| CLI | `typer` | Fast CLI building |
| Testing | `pytest` + `pytest-asyncio` | Standard |
| Linting | `ruff` | Fast |

## Project Structure

```
context-bridge/
├── src/
│   └── context_bridge/
│       ├── __init__.py
│       ├── server.py          # MCP server entrypoint
│       ├── router.py          # Request dispatcher
│       ├── config.py          # Pydantic settings
│       ├── cache.py           # TTL cache wrapper
│       ├── connectors/
│       │   ├── __init__.py
│       │   ├── base.py        # Connector ABC
│       │   ├── filesystem.py
│       │   ├── github.py
│       │   ├── ssh.py
│       │   └── obsidian.py
│       └── cli.py             # Typer CLI
├── tests/
│   ├── test_connectors/
│   └── test_router.py
├── docs/
│   ├── DESIGN.md
│   └── ARCHITECTURE.md
├── pyproject.toml
├── README.md
└── .gitignore
```

## Security Considerations

1. **Local Only by Default**: Server binds to localhost only
2. **No Credential Logging**: API tokens and SSH keys never logged
3. **Path Traversal Prevention**: All filesystem paths resolved and validated against configured roots
4. **SSH Key Permissions**: Warn if keys are too permissive (chmod 600 check)
5. **Command Allowlisting** (future): For SSH connector, optionally restrict allowed commands

## Performance Targets

- Tool call latency (cached): <10ms
- Tool call latency (uncached filesystem): <50ms
- Tool call latency (GitHub API): <2s
- File watch → cache invalidation: <100ms
- Startup time (load config + init connectors): <2s
- Memory footprint (idle): <100MB

## Future Extensions

1. **Vector Search**: Integrate with `sentence-transformers` + `faiss` for semantic retrieval
2. **Write Operations**: Allow LLM to create GitHub issues, edit files (with approval)
3. **Database Connector**: PostgreSQL, SQLite schema introspection
4. **Browser Connector**: Read open browser tabs (via extension or CDP)
5. **Docker Connector**: Inspect running containers, read logs


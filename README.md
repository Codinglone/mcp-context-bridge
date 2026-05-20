# Context Bridge

A unified context layer that connects your local data — repositories, documents, remote machines, and notes — to LLM interfaces through the Model Context Protocol (MCP).

## The Problem

Modern LLMs are powerful but context-starved. They don't know about:
- Your local codebase structure and recent changes
- Your Obsidian notes and personal knowledge base
- Your remote servers and their configurations
- Your project's documentation and dependencies
- Your PostgreSQL database schemas and query history
- Your Docker container logs and running services

You end up copy-pasting snippets or trying to explain complex contexts in a chat window.

## The Solution

Context Bridge runs locally, reads your data sources, and exposes them to any MCP-compatible LLM client (Claude Desktop, Cursor, Continue, Aether, etc.). It handles authentication, caching, chunking, and real-time updates so the LLM always has relevant context.

## Key Features

- **Filesystem Connector**: Watch local directories, respect `.gitignore`, provide file contents and tree structure
- **GitHub Connector**: Fetch repository code, issues, PRs, and discussions via API
- **SSH Connector**: Execute commands and read files on remote VMs/servers
- **Obsidian Connector**: Index your vault, follow wiki-links, search by tags and backlinks
- **PostgreSQL Connector**: Introspect schemas, list tables, view recent query history
- **Docker Connector**: Inspect running containers, stream logs, view service status
- **MCP Server**: Standard MCP protocol — works with any compatible client
- **Smart Chunking**: RAG-style retrieval so you don't burn context windows on irrelevant data
- **Real-time Sync**: File watchers push updates to connected clients automatically

## Status

🚧 Design phase. See `docs/DESIGN.md` and `docs/ARCHITECTURE.md` for current thinking.

## License

MIT

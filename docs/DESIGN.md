# Context Bridge — Design Document

**Version:** 0.1.0  
**Date:** 2026-05-20  
**Status:** Draft — pending review  

## 1. Purpose

Context Bridge is a local context provider for LLMs. It aggregates data from disparate sources (filesystem, GitHub, SSH, Obsidian, PostgreSQL, Docker) and serves it to MCP-compatible clients. The goal is to give LLMs the same situational awareness a human developer has when sitting at their machine.

## 2. Goals

1. **Unified Access**: One interface for all context sources
2. **Privacy First**: All data stays local unless explicitly sent to a remote LLM API
3. **Real-time**: Changes to local files are reflected in LLM context within seconds
4. **Efficient**: Don't dump 100k tokens of irrelevant data. Retrieve what matters.
5. **Pluggable**: New connectors should be easy to add

## 3. Non-Goals

1. **Not an LLM**: We don't host models. We feed context to existing ones.
2. **Not a RAG database**: We don't replace vector DBs; we complement them with live, structured context.
3. **Not a code editor**: No editing capabilities. Read-only context provision.
4. **Not an anonymization proxy**: We won't promise to strip all PII before sending to cloud LLMs. That's a separate hard problem.

## 4. User Stories

### Story 1: Local Codebase Context
> As a developer, I want my LLM to know my project's structure and recent git changes, so it can suggest fixes that match my patterns.

**Acceptance:**
- I can point Context Bridge at a local repo
- It provides file tree, recent commits, and file contents on request
- The LLM references my actual function names and imports

### Story 2: Remote Server Debugging
> As a DevOps engineer, I want my LLM to see logs and configs from my remote server without me SSHing and copy-pasting.

**Acceptance:**
- I configure an SSH connection with a key
- The LLM can request specific files or command outputs
- Credentials stay local; only the data I choose is transmitted

### Story 3: Personal Knowledge Base
> As a researcher, I want my LLM to reference my Obsidian notes when answering questions.

**Acceptance:**
- Context Bridge indexes my vault (respecting exclusions)
- The LLM can query by tag, link, or full-text search
- Backlinks and graph structure are preserved in context

### Story 4: GitHub Integration
> As an open-source maintainer, I want my LLM to review PRs with full issue context.

**Acceptance:**
- OAuth or token-based GitHub auth
- Fetch PR diffs, related issues, and CI status
- Provide this context alongside local code context

### Story 5: Database Context
> As a backend developer, I want my LLM to understand my PostgreSQL schema so it can write correct migrations and queries.

**Acceptance:**
- Configure connection string for local or remote PostgreSQL
- LLM can list tables, view schemas, and see recent query history
- Passwords never leave the local machine

### Story 6: Docker Debugging
> As a platform engineer, I want my LLM to see container logs and running services so it can diagnose issues without me running `docker ps` manually.

**Acceptance:**
- Context Bridge reads from local Docker socket
- LLM can get logs from specific containers or services
- Inspect container env vars, ports, and health status

## 5. Scope

### In Scope (v1)
- MCP server implementation (stdio and HTTP/SSE)
- Filesystem connector with `watchdog` monitoring
- GitHub connector (repos, issues, PRs)
- SSH connector (commands + file reads)
- Obsidian connector (vault indexing)
- PostgreSQL connector (schema introspection, query history)
- Docker connector (logs, container inspection)
- Simple CLI for configuration
- In-memory caching with TTL

### Out of Scope (v1)
- Vector embeddings / semantic search (use external RAG)
- Write operations (editing files, creating PRs)
- Web UI (CLI only)
- Multi-user / team features
- Cloud hosting

## 6. Success Criteria

1. A developer can install Context Bridge, point it at 2 repos, 1 Obsidian vault, and 1 PostgreSQL database, and query them from Claude Desktop within 10 minutes.
2. File changes are reflected in context within 5 seconds.
3. SSH command output and Docker logs can be retrieved and referenced by the LLM in a single conversation turn.
4. The system handles repos up to 100k files without crashing (via lazy loading and filtering).
5. PostgreSQL schema introspection returns table structures for databases with 100+ tables in under 3 seconds.

## 7. Open Questions

1. Should we support the new MCP Streamable HTTP transport, or just stdio + SSE?
2. What's the chunking strategy for large files (e.g., 10MB log files)?
3. How do we handle SSH connections that require interactive auth (2FA, passwords)?
4. Should GitHub connector cache aggressively, or fetch fresh data every time?


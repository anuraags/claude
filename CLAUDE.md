# Code Exploration

- Whenever you need to explore or understand source code in any repository, use the codebase memory MCP tools (e.g. `search_graph`, `trace_path`, `get_code_snippet`, `query_graph`, `get_architecture`, `search_code`) rather than relying on plain text search or file reads alone.
- Before exploring, confirm the repository is indexed (use `index_status`). If the repository has not been indexed yet, stop and prompt me to index it first — do not proceed with exploration until it is indexed.
- Grep/Glob/Read may still be used freely for non-code files (configs, docs, markdown), and you must always Read a file before editing it.

# Jest test structure

Follow these conventions when writing or refactoring Jest tests:

- **Self-contained files.** Each test file defines only the fixtures and mocks it
  needs. Do not share a combined fixture file across test files.
- **Mock setup in `beforeEach`.** Every test file has a `beforeEach` at the top that
  sets up all the mocks as necessary. A mock function's implementation (return value,
  resolved/rejected value, etc.) must be configured in this `beforeEach` block — never
  inline at the point where the mock function is created (e.g. not in the `jest.mock`
  factory or the `jest.fn(() => ...)` call itself).
- **Green path first.** Each test suite begins with a single green-path (happy path)
  test. Every subsequent test covers a path away from the green path (errors, edge
  cases, alternative branches).
- **Conditions go in `describe`, not `it`.** For a test that exercises a condition,
  wrap it in a `describe` block naming the condition, with the test itself in an `it`
  block inside. An `it` block must never contain a conditional.


<!-- BEGIN: apps-ecosystem-mcp -->
<!-- # Global Claude Instructions — Apps Team

## Session start — MCP health check

At the start of every session, before responding to any request, verify both MCPs are reachable:

1. Call `get_ecosystem_overview` on `apps-ecosystem-mcp`
2. Call `apps_get_registry` on `apps-actions-mcp`

For each MCP that fails or is unreachable, warn the user:
> "⚠️ `[mcp-name]` is disconnected. Make sure the server is running (`npm run docker:up` for apps-actions-mcp). I'll continue but functionality that depends on this MCP will not work."

On every subsequent request in the session, if an MCP was found disconnected at start, prepend a short reminder:
> "⚠️ `[mcp-name]` is still disconnected — connect it before this task if it relies on it."

## Rules

- The MCP health check runs **once per session only** — on the very first request. Do not repeat it on subsequent turns or skill invocations within the same session. -->
<!-- END: apps-ecosystem-mcp -->

# Prompt Conciseness

Reply in the most concise form possible. Skip pleasantries, preambles, and recaps of my question. No phrases like "I'd be happy to", "Great question", or "Let me explain". Drop articles and filler words wherever the meaning stays clear. Prefer short declarative sentences. If a tool call is needed, run it first and show only the result. Do not narrate your steps.

# Code Exploration

- Whenever you need to explore or understand source code in any repository, use the codebase memory MCP tools (e.g. `search_graph`, `trace_path`, `get_code_snippet`, `query_graph`, `get_architecture`, `search_code`) rather than relying on plain text search or file reads alone.
- Before exploring, confirm the repository is indexed (use `index_status`). If the repository has not been indexed yet, stop and prompt me to index it first — do not proceed with exploration until it is indexed.
- Grep/Glob/Read may still be used freely for non-code files (configs, docs, markdown), and you must always Read a file before editing it.

# Code Comments

- Keep comments very concise and specific. Prefer clean, self-explanatory code over comments that clarify what the code does.
- Only leave a comment when logic is particularly tricky. If you think something needs a comment, ask me first.
- Do not put JIRA tickets in comments — those belong in the commit message.

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



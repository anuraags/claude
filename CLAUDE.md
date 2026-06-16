# Jest test structure

Follow these conventions when writing or refactoring Jest tests:

- **Self-contained files.** Each test file defines only the fixtures and mocks it
  needs. Do not share a combined fixture file across test files.
- **Mock setup in `beforeEach`.** Every test file has a `beforeEach` at the top that
  sets up all the mocks as necessary.
- **Green path first.** Each test suite begins with a single green-path (happy path)
  test. Every subsequent test covers a path away from the green path (errors, edge
  cases, alternative branches).
- **Conditions go in `describe`, not `it`.** For a test that exercises a condition,
  wrap it in a `describe` block naming the condition, with the test itself in an `it`
  block inside. An `it` block must never contain a conditional.

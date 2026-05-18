# C++ Testing Principles

Covers test intent, Catch2 construct selection, and keeping test bodies focused on behavior.

## Framework

The patterns here assume Catch2 with `catch_discover_tests`, so each `TEST_CASE` shows up as its own CTest entry. Most guidance transfers to GoogleTest, doctest, or Boost.Test with equivalent constructs. Catch2-specific guidance is labeled.

## Core principle

A test verifies the contract, invariants, and expected outcomes of the code. See `philosophy.md` for the full rationale.

## Navigation

Start with `philosophy.md`; the rest are reference.

| File                       | Covers                                                                  |
|----------------------------|-------------------------------------------------------------------------|
| `philosophy.md`            | Testing behavior over implementation, understanding intent, component vs integration scoping. |
| `catch2-conventions.md`    | Catch2/CTest discovery, serialization, tags, assertions.                |
| `test-patterns.md`         | Choosing between `TEST_CASE`, `SECTION`, `GENERATE`, `TEMPLATE_TEST_CASE`; table-driven tests. |
| `test-helpers.md`          | Factory functions, test data conventions, fixtures, probes, providers.  |
| `error-path-testing.md`    | Testing success and failure paths in result-style and exception-throwing code. |
| `condition-based-waiting.md` | Predicate-based waits for async operations.                         |
| `approval-tests.md`        | Conformance tests with ApprovalTests.                                   |
| `qt-gui.md`                | Qt-specific testing: offscreen platform, `QSignalSpy`, model/view, object lifetime. |

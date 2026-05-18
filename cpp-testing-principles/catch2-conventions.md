# Catch2 and CTest conventions

This document defines how Catch2 cases are registered with CTest, how tags are
spelled, and which assertion macros to use.

## Snippet conventions

Examples assume Catch2 v3. Use angle-bracket includes, `snake_case` test
names, lowercase bracketed tags, and C++20 designated initializers with
`.field = value`.

## Test discovery and selection

`catch_discover_tests` registers each `TEST_CASE` as its own CTest entry.
`TEST_PREFIX` prepends a string to the registered names, which scopes CTest
filters to a module:

Without `TEST_PREFIX`, every registered name is just the bare scenario name:

```
parser_test -- "parses empty input"
container_test -- "parses empty input"   # same name, different binary
```

With `TEST_PREFIX "<module>/"`, the origin is part of the name:

```cmake
catch_discover_tests(<module>_test
  TEST_PREFIX "<module>/"
)
```

```
parser/parses empty input
container/parses empty input
```

With the prefix in place, `ctest -R "<module>/"` runs only that module's tests, and each CTest entry carries its origin in its name -- which prevents false matches when scenario names recur across modules.

CTest filtering is name-based. Tag-based filtering lives one layer down, in the binary itself: `./<module>_test "[parser]"` selects by tag. `TEST_SPEC` is what surfaces a tag-selected subset as its own CTest entry -- the serialization snippet below uses it to give `[serial]` tests their own `RUN_SERIAL` CTest properties.

## Test serialization

Tests that cannot run in parallel -- global initialization, network or
filesystem usage, process-wide singletons, anything that touches shared
mutable state -- need to be marked `RUN_SERIAL` on the CTest side so the
runner does not schedule them concurrently with other tests in the same
binary:

```cmake
catch_discover_tests(<module>_test
  TEST_SPEC "[serial]"
  PROPERTIES RUN_SERIAL TRUE
)
```

## Tags

Tags pick test subsets at the command line. Use lowercase, one bracketed token per axis: `[module][component][feature]`.

- Module: `[container]`, `[parser]`, `[cache]`.
- Component: `[ring_buffer]`, `[lexer]`.
- Feature/trait: `[roundtrip]`, `[serial]`, `[slow]`.

Inconsistent tag spelling within a module causes `--list-tags` to show duplicates that filter differently than expected.

## Assertions

- `Approx` requires `#include <catch2/catch_approx.hpp>` -- it is not pulled in by the default Catch2 header.
- `CAPTURE(var)` is essential inside generators and table-driven tests so the failure message identifies the failing row.

`REQUIRE` is the right call when the assertion is a precondition for
anything that follows: if a parse was supposed to succeed and did not, the
rest of the test cannot run meaningfully. `CHECK` is the right call when
several independent observations should all hold and the test should report
every failure, not just the first.

## See also

- `test-patterns.md` -- choosing between `TEST_CASE`, `SECTION`,
  generators, and typed tests.
- `test-helpers.md` -- keeping setup readable without hiding behavior.

# Root-cause tracing

Most bugs surface where the code finally tries to use a bad value -- a
parser rejects an empty field, a connector refuses an invalid handle, an
assertion catches a violated invariant. The instinct is to fix it where the
error message points. That is the symptom, not the cause.

**Core principle:** trace the bad value backward through the call chain to
where it originated, fix it at the root cause, and add validation at the layers
in between.

## When to trace backward

Tracing pays off when:

- The error appears deep in the call stack, far from the entry point.
- The stack trace shows a long call chain and it is not yet clear which
  frame introduced the problem.
- It is unclear where invalid data originated.
- You need to identify which test, request, or input triggers the problem.

If the error already surfaces at the entry point, fix it there.

## Tracing procedure

### 1. Observe the symptom precisely

Capture the failure exactly: file, line, error message, and the value that
provoked it. Replace "sometimes it fails" with "`router::lookup` rejects an
empty key at the lookup line". Only the precise version can be traced.

```text
ERROR: invalid_argument in router.cpp:245 - empty key in routing table lookup
```

### 2. Find the immediate cause

Read the code that directly produces the error.

```cpp
auto router::lookup(std::string_view key)
    -> std::expected<destination, error_code>
{
  if (key.empty()) {
    return std::unexpected{error_code::invalid_argument};
  }
  // ...
}
```

The function is doing its job: it rejects an empty key. The interesting
question is who passed it the empty key.

### 3. Walk one level up

What is the caller, and what did it pass?

```cpp
auto dispatcher::dispatch(message m) -> std::expected<void, error_code>
{
  auto destination = router_.lookup(m.routing_key);  // empty key arrived here
  // ...
}
```

The dispatcher passed an empty `routing_key`. Where did `m` come from?

### 4. Keep tracing

```cpp
auto session::on_message(buffer b) -> void
{
  auto msg = parse(b);
  dispatcher_.dispatch(msg);   // parse produced the empty key
}
```

The message came from `parse(b)`. Look at `parse`.

```cpp
auto parse(buffer b) -> message
{
  message m;
  m.payload = std::string{b.data(), b.size()};
  // routing_key keeps its default empty value
  return m;
}
```

### 5. Identify the root cause

`parse` returns a message with `routing_key` left at its default. The
default-initialized empty string survives all the way to the lookup, which
rejects it. The cause is not in the router; it is in the parser that
forgot to populate the field. The eventual fix should make `parse` return a
result so malformed input stops at the boundary.

### 6. Fix at the root cause, then reinforce the path

The root cause is in `parse`. The fix belongs there: either extract the
routing key from the buffer or return an explicit error when the buffer does
not contain one. Then use `defense-in-depth.md` to decide which intermediate
layers should reject the invalid value explicitly.

## Instrumenting when manual tracing runs out

When the call chain is long, branches across callbacks, or crosses a thread
boundary, manual reading is no longer enough. Add an instrumented log at the
failing point that captures a stack trace, reproduce the failure, and read
the trace.

```cpp
#include <cpptrace/cpptrace.hpp>

auto router::lookup(std::string_view key)
    -> std::expected<destination, error_code>
{
  if (key.empty()) {
    log_debug("router::lookup: empty key\n{}",
              cpptrace::generate_trace().to_string());
    return std::unexpected{error_code::invalid_argument};
  }
  // ...
}
```

C++23 ships `<stacktrace>` in the standard library; `cpptrace` is a
header-only alternative that works on earlier standards.

The trace pinpoints the path that reached the failure -- which test, which
caller, which branch. From there, walking up is mechanical.

For builds without runtime stack-trace support, attach `gdb` or `lldb`,
set a breakpoint at the failing line, and inspect the call stack when it
fires.

## Log before the failure, not after

A log at the point of failure tells you that the failure happened. A log
*before* the failing operation tells you the state at the moment the
decision was made, which is what tracing needs.

```cpp
// Capture state before the call that might fail.
log_debug("dispatch: id={}, routing_key='{}', payload_size={}",
          m.id, m.routing_key, m.payload.size());
auto destination = router_.lookup(m.routing_key);
```

Test runners often suppress the production logger; use `std::cerr` so the
trace reaches test output.

## Finding the test that pollutes state

This is a bisection problem, not a backward-tracing problem: you do not
follow a value through a call chain; you narrow down which test in a
suite creates unwanted state. The technique is otherwise mechanical.

Confirm the unwanted state is absent before the run, then bisect: run
half the tests, check; if clean, run the other half; otherwise narrow
into the failing half. Each step halves the search.

A shell loop handles small suites directly:

```bash
# Adapt to the test runner. Adjust the path being checked.
for test in tests/*; do
  rm -f path/to/unwanted_state
  run-test "$test" >/dev/null 2>&1 || true
  if [ -e path/to/unwanted_state ]; then
    echo "POLLUTER: $test"
    break
  fi
done
```

The same pattern applies to non-filesystem pollution: an open socket, a
process-wide singleton, an entry in a shared registry. Replace the `-e`
check with whatever query inspects the relevant state.

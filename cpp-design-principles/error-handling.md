# Error handling

Two result types cover most fallible operations in modern C++:
`lib::result<T>` and `std::expected<T, E>`. They are not interchangeable,
and the choice depends on where the error has to travel. In these examples,
`lib::result<T>` is the project alias for `boost::leaf::result<T>`.

For the corresponding test patterns -- asserting on success and failure
returns, matching on structured error payloads, and verifying scope-guard
rollback -- see `../cpp-testing-principles/error-path-testing.md`.

LEAF should stay contained inside a domain. Inside a domain it
carries rich, structured context cheaply and lets handlers match on
whatever fields they care about. At domain boundaries, convert to a result
the caller's vocabulary supports. Returning `lib::result<T>` across unrelated
boundaries forces callers into the LEAF machinery unnecessarily.

Code at a domain boundary should call `leaf::try_handle_*` and convert the
captured failure into a shape the caller can consume without seeing LEAF.
The shape is picked case by case:

- **void**, when the failure is handled in place -- logged, retried,
  delivered through a callback, or translated into another cross-boundary
  signal with its own meaning.
- **`std::optional<T>`**, when "no value" is a sufficient summary and the
  caller does not need a reason.
- **`std::expected<T, E>`**, when the caller needs to react differently to
  different errors. `E` is the consuming domain's error type, not a LEAF
  carrier.

- **Errors live in the domain that produces them.** Each domain owns its
  error codes (typically an `error_code.hpp` with an `enum class`) and any
  structured error payloads (`errors.hpp`). Code in domain `A` returns
  `A`'s errors.
- **Errors do not cross unrelated domain boundaries.** An adapter that sits
  between domain `A` and domain `B` may handle both error families, because
  it directly depends on both. A consumer of `A` should not have to know
  about `B`'s error codes to interpret a failure.

```
adapter_a_b        Can produce and consume A's and B's errors (direct deps)
   |    \
   A     B         Cannot see each other's error codes (unrelated domains)
```

If two domains repeatedly translate each other's errors, an adapter is missing. The same shape applies to data flow;
see "Domain separation with adapters" in `architecture.md`.

## Domain error types

A domain describes its failure surface the same way it describes its
data surface: explicit, owned, and visible at the boundary. Two small
headers do the work:

- `error_code.hpp` -- an `enum class error_code` whose enumerators name
  every distinct failure the domain can produce.
- `errors.hpp` -- structured error types holding the context that
  matters for each kind of failure: the offending field or order id.

Both halves earn their keep. The code is a stable, comparable
identifier suitable for `std::error_code`, log lines, and metrics. The
structured type lets a handler reach the failed field or id directly,
without parsing a message string back out.

This mirrors the convention for data types -- a domain owns its types
(see `compile-time-correctness.md`) and a domain owns its errors. The reasoning is the
same in both directions: the producer is the one with the context to
name things accurately, and consumers should not invent their own
vocabulary for the producer's concepts.

### Codes integrate with `std::error_code`

Each domain's `error_code` enum registers a custom
`std::error_category`, so the enum implicitly converts to
`std::error_code`, `message()` produces the same text everywhere, and
comparisons work across domains. Logs, metrics, and persisted records
all speak the standard interface.

The plumbing is identical for every domain: a category subclass, a
`make_error_code(error_code)` free function in the domain namespace,
and a `std::is_error_code_enum` specialization. A single macro
generates all three, so each domain writes one line:

```cpp
// in domain/error_code.hpp
namespace domain {

// Each domain reserves a numeric prefix (here, DOMAIN_PREFIX) so codes
// stay globally unique across the program. The trailing digits enumerate
// the failures within the domain.
enum class error_code : std::int32_t {
  invalid_field_value    = DOMAIN_PREFIX + 1,
  missing_required_field = DOMAIN_PREFIX + 2,
  unknown_order          = DOMAIN_PREFIX + 3,
};

constexpr const char* to_string(error_code ec) {
  switch (ec) {
    case error_code::invalid_field_value:    return "invalid field value";
    case error_code::missing_required_field: return "missing required field";
    case error_code::unknown_order:          return "unknown order";
  }
  return "unknown error";
}

}  // namespace domain

LIB_DEFINE_ERROR_CATEGORY(domain)
```

Numbering enumerators with a per-domain numeric prefix keeps codes globally unique. A small static checker
can verify uniqueness, that every code has a `to_string` arm, and that
the prefix matches the domain's assigned range.

### Structured errors carry typed context

A code names the failure; the struct names which order, field, or value.

```cpp
// in domain/errors.hpp
namespace domain::errors {

struct invalid_field_value {
  types::field_id field;
  types::text value;
  std::optional<std::string> text{};

  std::error_code code() const {
    return make_error_code(domain::error_code::invalid_field_value);
  }

  std::string what() const {
    return fmt::format("'{}' for field '{}'", value, field);
  }
};

}  // namespace domain::errors
```

The two methods, `code()` and `what()`, are the only shape these
structs share. A concept makes the contract explicit:

```cpp
template <typename T>
concept ErrorData = requires (T t) {
  { t.code() } -> std::same_as<std::error_code>;
  { t.what() } -> std::same_as<std::string>;
};
```

Handlers that want the typed context reach for the struct's fields;
handlers that only need the code or the message go through the
concept methods.

### A type-erased carrier propagates through LEAF

`lib::result<T>` is parameterized only on the success type; the
error rides as a side channel. A small type-erased `lib::error` holds
any `ErrorData` value, exposes the two concept methods, and lets
handlers cast back to the structured payload:

```cpp
class error {
public:
  template <ErrorData T>
  explicit error(T&& value, /* options, source_location */);

  std::error_code code() const;
  std::string what() const;

  template <ErrorData T> bool is_type() const;
  template <ErrorData T> T* data();
  // ...
};
```

Intermediate code stays uniform -- every fallible function returns
`lib::result<T>`, no template parameter on the error type explodes
across the codebase, no `std::variant<E1, E2, ...>` has to be threaded
through. The failure site constructs a typed struct via `lib::new_error`;
a handler at the boundary casts back to it; everything between treats
the error as opaque.

### Handlers match on the structured type

LEAF dispatches to a tuple of handlers, picking the first whose
argument type matches the active error. A pair of predicate types --
`match_error<T>` for a single domain struct, `match_errors<T...>` for
several -- inspects the carried `lib::error` and gives the handler
typed access to the payload:

```cpp
boost::leaf::try_handle_all(
  [&]() -> lib::result<reply> {
    BOOST_LEAF_ASSIGN(auto routed, router.route(request));
    BOOST_LEAF_ASSIGN(auto sent,   session.send(routed));
    return build_reply(sent);
  },

  // single-type match: typed access to the offending field
  [](lib::match_error<domain::errors::invalid_field_value> m) -> reply {
    return make_reject(m.value().field, m.value().value);
  },

  // multi-type match: same translation for a family of codes
  [](lib::match_errors<domain::errors::duplicate_order,
                       domain::errors::duplicate_request> m) -> reply {
    return drop_silently(m.matched);
  },

  // catch-all for anything not matched above
  LIB_RESULT_CATCH_ALL(return internal_error_reply();)
);
```

The tuple is ordered: LEAF tries handlers top to bottom and stops at
the first match. A useful convention is:

1. **Preconditions first** -- duplicates and other "the same request
   twice" cases, typically logged and dropped.
2. **Domain-specific translations** -- typed errors mapped to the
   caller's reply.
3. **Default handler for a family of codes** -- a `match_errors<...>`
   covering the codes that all reduce to "generic reject with this
   code and message".
4. **Catch-all** -- logs the unmatched error and returns a safe
   fallback so no failure escapes silently.

When the same shape repeats across request types, a small factory
builds the tuple once and the top-level function asks for handlers
parameterized on the request it is processing. The dispatched
handlers close over the request and translate into the reply type it
expects.

The `ErrorData` concept connects both ends: handlers match on the typed
struct; everything between treats the error as opaque.

## Exceptions do not cross component boundaries

A component's public surface is a contract about how it can fail. A
function that returns `lib::result<T>` or `std::expected<T, E>`
advertises every failure it can produce; the caller does not also
have to be ready for an unrelated exception to come up through it.
The same applies when the component invokes a callback the caller
registered with it -- the component must not let an exception of its
own escape through that callback frame, where the caller has no way
to know which exception types are even possible.

The rule runs in both directions:

- **Calls into the component from outside** -- request handlers,
  message receivers, timer ticks, callback entry points -- catch every
  exception before returning to the caller.
- **Calls the component makes to its own dependencies** -- effectful
  operations on state it owns, fallible adapters, third-party
  libraries that throw -- have their exceptions caught at the
  component boundary, not allowed to unwind through code in the
  component that did not opt into them.

Registered callbacks run on the caller's policy. Dispatch them outside
`try_handle_all` so a throw lands in the caller's frame. The dispatching
side does not have information or context to be able to handle exceptions
thrown inside the callback handler.

The practical pattern is to bracket the component's own work in a
single `try_handle_all` and dispatch to registered callbacks
*outside* that block, on its result. A throw out of a callback then
escapes to the caller's frame, where it belongs:

```cpp
void router::send(request&& req)
{
  // The router's own work -- target selection, child request tracking,
  // parent request processing -- runs inside try_handle_all. Exceptions
  // from this code are the router's to catch.
  auto result = boost::leaf::try_handle_all(
    [&]() -> lib::result<reply> {
      // ... router logic ...
    },
    domain::make_error_handlers(req));

  // Callbacks run on the caller's stack frame; their exceptions
  // propagate to router::send's caller.
  if (result) {
    on_request(std::move(*result));
  } else if (result.error()) {
    on_reject(std::move(*result.error()));
  }
}
```

The narrow exception is a utility library whose contract explicitly
documents that some operations throw -- `std::stoi`, a container's
`at()`, a parser that throws on malformed input. A component may let
such throws propagate internally, catching and translating them before
they reach the component's own public boundary. An exception escaping
that boundary is a bug.

### Top-level functions catch everything

A *top-level function* is any function whose stack frame is the first
one above the runtime: a request handler called from an I/O thread, a
callback registered with an event loop, a worker pulling off a queue,
a timer tick, a lifecycle hook (start, poll, stop) the host invokes
on the component. There is no application-level frame above it to
catch what it lets escape -- only `main`'s implicit handler, which
terminates the process.

A resilient program does not depend on `main()` wrapping the world in
`try`/`catch`. Each top-level function takes responsibility for
catching what it produces, logging it, and handing control back to the
runtime cleanly. The next request still gets handled; one bad message
does not take the process down.

`leaf::try_handle_all` is the right shape for the catch because it
captures **both** failures returned through `lib::result<T>` and
exceptions thrown anywhere in the call tree -- including ones thrown
from inside code that itself returns `lib::result<T>`. The body of
the function can use `BOOST_LEAF_CHECK` / `BOOST_LEAF_ASSIGN` freely,
and a stray `throw` from deep inside still lands in the same handler.

```cpp
// BAD - exception from any leg escapes to the runtime
void session::receive(message&& msg)
{
  auto state = orders_.process(msg);    // may throw
  if (auto reply = build_reply(state)) {
    source_.send(*reply);               // may throw
  }
}
```

```cpp
// GOOD - top-level function: every failure path (result error,
//        thrown exception, out-of-memory) is captured here
void session::receive(message&& msg)
{
  boost::leaf::try_handle_all(
    [&]() -> lib::result<void> {
      BOOST_LEAF_ASSIGN(auto state, orders_.process(msg));
      BOOST_LEAF_CHECK(source_.send(build_reply(state)));
      return {};
    },
    on_structured_error,   // typed error payloads carried by LEAF
    on_std_exception,      // anything deriving from std::exception
    on_unknown);           // catch-all for everything else
}
```

The handlers log, increment a metric, optionally surface the failure
through a callback the component exposes for that purpose, and
return. The runtime never sees the exception.

Internal helpers underneath a top-level function return
`lib::result<T>` like any other fallible function; both their
explicit errors and any exceptions they let propagate flow up to
the single `try_handle_all` at the boundary.

## Enriching errors with diagnostic context

`lib::error` captures a `std::source_location` from every `new_error`
call site and, when enabled, a stack trace via `cpptrace` or
`boost::stacktrace`. Stack-trace capture is opt-in -- a compile-time
flag for debug builds and a per-call option for specific paths -- so
release builds pay no cost unless they ask. For the full details on
attaching context at the failure site, capturing traces from caught
exceptions, and the debug/release toggle, see
`../cpp-debugging-principles/root-cause-tracing.md`.

## See also

- `invariants.md` -- the rollback patterns there compose with both
  `lib::result` and thrown exceptions; scope guards fire on either
  exit path, so a top-level `try_handle_all` does not have to know
  which kind of failure occurred to leave state consistent.
- `architecture.md` -- "Functional core, imperative shell": the
  shell is where top-level functions live, and it is the natural
  home for `try_handle_all` blocks that separate the failure-free
  inner layers from the runtime's expectations.

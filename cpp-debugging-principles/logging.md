# Logging

A logging facility is the cross-cutting service every layer reaches for.
The rules below shape the call surface so disabled levels cost nothing,
hot call sites cannot drown the sink, and tests can capture or silence
output without rebuilding.

## Severity family

Ship six severities: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`,
`CRITICAL`. The split between `TRACE` and `DEBUG` is the one that
matters in practice: `TRACE` is the level that captures hot-path detail,
and it stays compiled out in release builds by default. `DEBUG` is for
the slower paths that are still worth seeing in a development build but
do not belong in a release.

## Compile-time level cutoff

Calls below the active level must collapse to `(void)0` so disabled
levels have zero release cost. The active level is a compile-time
constant the build chooses per preset, not a runtime variable read on
every call. Default to `INFO` in release builds, `DEBUG` in development,
and offer a `releaseWithTrace`-style preset that bumps the active level
to `TRACE` for capture sessions without otherwise disturbing the release
configuration.

```cpp
// macro shape - level is selected at compile time, so disabled levels
// expand to (void)0 and incur no runtime work
#if LIB_LOG_ACTIVE_LEVEL <= LIB_LOG_LEVEL_TRACE
#  define LIB_LOG_TRACE(...) lib::log::write(lib::log::level::trace, __VA_ARGS__)
#else
#  define LIB_LOG_TRACE(...) ((void)0)
#endif
```

## Throttled variants for hot call sites

A `_THROTTLED(interval, ...)` form per severity keeps a noisy log call
from drowning the sink. The throttler keys on call-site location, not on
the message string, so the rate limit follows the source file and line
rather than the formatted output. The throttling decision is made before
formatting; suppressed calls do not pay the format cost.

```cpp
LIB_LOG_WARN_THROTTLED(std::chrono::seconds{1},
                       "queue depth above watermark: {}", depth);
```

## Formatter support for domain types

Log formatting goes through `fmt`. Project formatters cover strong
types, optionals, error codes, and the domain enums every component
already uses, so a call site never has to call `.get()` or `to_string()`
just to log a value. A call site that has to unwrap a strong type to log
it is a missing formatter, not a logging quirk.

```cpp
// GOOD - the strong type, the optional, and the error code all format directly
LIB_LOG_INFO("order {} settled with reason {}", order_id, settle_reason);
```

```cpp
// BAD - unwrap noise around a logger that does not know the domain types
LIB_LOG_INFO("order {} settled with reason {}",
             order_id.get(), to_string(settle_reason));
```

## Logger as a variant-backed cross-cutting service

The logger is the canonical cross-cutting service. Backends are
alternatives of a `std::variant` -- at minimum `console_logger`,
`file_logger`, and `null_logger` -- composed near `main` by emplacing
the chosen alternative. Tests install `null_logger` (or a capturing
backend) through an RAII restore guard so the next test starts from a
known state. See `../cpp-design-principles/cross-cutting.md` for the
full pattern.

```cpp
namespace logging {

using logger = std::variant<console_logger, file_logger, null_logger>;
inline logger global{null_logger{}};

}  // namespace logging
```

```cpp
// main.cpp - choose the backend once at startup
logging::global.emplace<logging::file_logger>(config.log_path);
```

The variant gives every layer the same dispatch shape -- `lib::match`
over `global` inside each free-function entry point -- without virtual
dispatch and without threading a logger handle through every
constructor.

# Cross-cutting services

A handful of services -- a clock, a logger, a timer, a metrics sink -- are
reachable from every layer of a program. Threading them through every
constructor distorts every signature in the codebase; routing them through a
polymorphic singleton trades that pain for vtable indirection and a chain of
inheritance to keep alive. A `std::variant<Impl1, Impl2, ...>` configured by
`main` at startup, with free functions that dispatch via `lib::match`, gives
the same swap-implementation property without virtual dispatch and without
constructor plumbing.

## The problem

Cross-cutting concerns are the services that any layer might reach for: read
the current time, emit a log line, schedule a callback, increment a counter.
They are not domain logic, and they do not belong to any one component, but
nearly every component has occasion to use them.

Two common solutions both fall short.

### Constructor injection through every layer

Pass a `clock&`, a `logger&`, and a `timer&` to every component that might
need them. The advertised benefit is testability: the test substitutes a
fake at construction time. The actual cost is felt at every layer between
the composition root and the leaf that uses the service. Constructors gain
parameters they only forward; member fields hold references they only pass
along; refactors that move a component up or down the tree have to thread
or unthread the same handles through dozens of intermediate signatures.

```cpp
// BAD - the parser does not use the clock; it forwards it so the
//       validator can forward it so the rule engine can read it
class parser {
public:
  parser(clock& time, logger& log) : time_{time}, log_{log} {}
  // ...
private:
  clock&  time_;
  logger& log_;
};
```

The signature lies: `parser` advertises a dependency on `clock` it does not
use. The reader has to chase the handle down to the layer that actually
calls `time.now()` to learn that.

### Polymorphic singleton

Define an abstract `clock` base, give it a `static clock& instance()`
accessor, and let `main` install a concrete subclass at startup. The
ergonomic problem disappears: any layer reaches the service through a free
function. Three other problems take its place.

- **Virtual dispatch on every call.** Reading the clock is a virtual call;
  writing a log line is a virtual call. The cost is small per call, but
  these are the highest-frequency services in the program.
- **Inheritance coupling.** Adding an alternative implementation means
  inheriting from the base, which means the base header is in every
  alternative's translation unit. A change to the interface ripples through
  every implementation.
- **One-definition-rule hazards under dynamic linking.** An inline template
  variable that holds the singleton can end up with one address per shared
  library, producing two coexisting "singletons" that disagree about state.
  Static-initialization order across libraries adds a second class of bug.

## The pattern

A header declares an inline `std::variant<Impl1, Impl2, ...>` global and a
small set of free functions that dispatch on it via `lib::match`. `main`
selects the alternative once, at startup, with `emplace<production_impl>(config)`.
Tests substitute by `emplace<mock_impl>()` in their setup. Dispatch is a
compile-time visitor; no virtual function table is involved.

```cpp
// logger.hpp
namespace logging {

struct console_logger {
  void log(level lvl, std::string_view message);
};

struct file_logger {
  std::filesystem::path path;
  void log(level lvl, std::string_view message);
};

struct null_logger {
  void log(level, std::string_view) noexcept {}
};

using logger = std::variant<console_logger, file_logger, null_logger>;

inline logger global{null_logger{}};

template <typename... Args>
void log(level lvl, fmt::format_string<Args...> format, Args&&... args)
{
  auto message = fmt::format(format, std::forward<Args>(args)...);
  lib::match(global, [&](auto& impl) { impl.log(lvl, message); });
}

}  // namespace logging
```

```cpp
// main.cpp
int main(int argc, char** argv)
{
  auto config = parse_arguments(argc, argv);
  logging::global.emplace<logging::file_logger>(config.log_path);
  // ... rest of startup ...
}
```

Three properties matter:

- **Single address.** `global` is an inline variable of a non-template type.
  It has one definition, one storage location, and one set of contents,
  even when several shared libraries include the header.
- **Compile-time dispatch.** `lib::match` resolves to a direct call on the
  active alternative. The compiler can inline through it the same way it
  inlines a normal function call.
- **Closed set, open swap.** The set of alternatives is fixed at compile
  time, but which alternative is active is decided at runtime by `main` (or
  by a test fixture). Swapping a fake in for a test is one line.

This is the cross-cutting analogue of `runtime.md`'s threading-at-the-edge
principle: the inner code calls the service through a free function; the
edge chooses which implementation it dispatches to. `pipelines.md` applies
the same shape to per-edge data flow, where the wiring lambda is the
chosen implementation.

## Why a variant, not an inline template variable

Ben Deane's "Using Types Effectively" talk (CppCon 2016, linked below)
sketches a related pattern built on an inline template variable. It works
in single-binary applications. With dynamic linking it stops working:
different translation units in different shared libraries can end up with
different addresses for what is supposed to be the one variable. Static
initialization order across libraries adds a second class of bug.

The variant form avoids both. The variant is a normal inline variable of a
non-template type, so there is one definition and one address. The
implementation alternatives are normal class types, not template
specializations, so they do not multiply across translation units. The
dispatch happens at the call site through `lib::match`, which is generated
once per call site and resolves through the variant's index, not through
template instantiation.

The trade is that the set of alternatives is closed: adding a new one means
recompiling code that includes the header. For cross-cutting services this
is a fair price, because the set is small (production, test, and one or two
sinks) and rarely changes.

Reference: Ben Deane, "Using Types Effectively," CppCon 2016 --
https://www.youtube.com/watch?v=ojZbFIQSdl8

## Trade-offs

The pattern earns its keep for a narrow band of services. The trade-offs
are worth being explicit about.

- **Closed set of implementations.** Adding a new alternative requires
  recompiling code that includes the header. For genuinely cross-cutting
  services -- clock, logger, timer, metrics -- the set is small and stable.
  For services where the set of implementations is open or evolving, prefer
  ordinary dependency injection.
- **Free-function dispatch.** Calling `log(level::info, "...")` is slightly
  less ergonomic than calling `logger_.info("...")` on an injected member.
  In return, every layer of the program is free of an unused logger handle.
- **Globals invite over-reach.** A global that is easy to call is easy to
  call from places that should have taken a parameter instead. Reserve the
  pattern for services that are genuinely cross-cutting; a domain
  dependency is not cross-cutting just because it is annoying to inject.
- **Startup ordering.** `main` is responsible for assigning a real
  implementation before any code reads the global. A safe default
  alternative (`null_logger`, a clock that returns the epoch) keeps an
  early read from misbehaving, but a meaningful program still has to
  install the real implementation before its first request lands.

## Test substitution

A test fixture swaps the alternative in its setup and the next fixture
swaps it back. No mocking framework is involved.

```cpp
namespace timekeeping {

struct mock_clock {
  std::chrono::system_clock::time_point now_value;

  auto now() const -> std::chrono::system_clock::time_point { return now_value; }
  void advance(std::chrono::milliseconds delta) { now_value += delta; }
};

}  // namespace timekeeping

struct clock_fixture {
  clock_fixture()
  {
    timekeeping::global.emplace<timekeeping::mock_clock>(
        timekeeping::mock_clock{.now_value = std::chrono::system_clock::time_point{}});
  }

  ~clock_fixture()
  {
    timekeeping::global.emplace<timekeeping::system_clock>();
  }

  auto mock() -> timekeeping::mock_clock&
  {
    return std::get<timekeeping::mock_clock>(timekeeping::global);
  }
};

TEST_CASE_METHOD(clock_fixture, "expiry fires after the timeout elapses")
{
  auto session = build_session();

  mock().advance(std::chrono::seconds{30});
  CHECK(session.expired());
}
```

The fixture replaces the global before the test body runs and restores a
default alternative on destruction, so the next test starts from a known
state. The body of the test reaches the mock through `std::get` to drive
it; production code in the same test path reads `timekeeping::now()` and
gets the mocked value through the same dispatch path.

For broader test-fixture conventions, see `../cpp-testing-principles/`.

## A small catalog

Three services side by side, all the same shape. Each declares the
variant, the alternatives, and the free functions that dispatch on it.

### Logger

```cpp
namespace logging {

struct console_logger { void log(level lvl, std::string_view message); };
struct file_logger    { std::filesystem::path path;
                        void log(level lvl, std::string_view message); };
struct null_logger    { void log(level, std::string_view) noexcept {} };

using logger = std::variant<console_logger, file_logger, null_logger>;
inline logger global{null_logger{}};

template <typename... Args>
void log(level lvl, fmt::format_string<Args...> format, Args&&... args)
{
  auto message = fmt::format(format, std::forward<Args>(args)...);
  lib::match(global, [&](auto& impl) { impl.log(lvl, message); });
}

}  // namespace logging
```

### Timer

```cpp
namespace timers {

struct event_loop_timer {
  event_loop* loop;
  auto schedule(std::chrono::milliseconds delay, std::function<void()> callback) -> handle;
  void cancel(handle h);
};

struct manual_timer {
  std::vector<scheduled_callback> pending;
  auto schedule(std::chrono::milliseconds delay, std::function<void()> callback) -> handle;
  void cancel(handle h);
};

using timer = std::variant<event_loop_timer, manual_timer>;
inline timer global{manual_timer{}};

inline auto schedule(std::chrono::milliseconds delay, std::function<void()> callback) -> handle
{
  return lib::match(global, [&](auto& impl) { return impl.schedule(delay, std::move(callback)); });
}

inline void cancel(handle h)
{
  lib::match(global, [&](auto& impl) { impl.cancel(h); });
}

}  // namespace timers
```

### Clock

```cpp
namespace timekeeping {

struct system_clock {
  auto now() const -> std::chrono::system_clock::time_point
  {
    return std::chrono::system_clock::now();
  }
};

struct mock_clock {
  std::chrono::system_clock::time_point now_value;
  auto now() const -> std::chrono::system_clock::time_point { return now_value; }
  void advance(std::chrono::milliseconds delta) { now_value += delta; }
};

using clock = std::variant<system_clock, mock_clock>;
inline clock global{system_clock{}};

inline auto now() -> std::chrono::system_clock::time_point
{
  return lib::match(global, [](const auto& impl) { return impl.now(); });
}

}  // namespace timekeeping
```

The shape repeats: a variant of small alternatives, an inline global, and
free functions that dispatch through `lib::match`. A reader who learns the
pattern once recognizes it on sight everywhere it appears.

## The match helper

`std::visit` accepts a single callable. `lib::match` wraps the variant and
the per-alternative callables into one expression:

```cpp
template <typename... Fs>
struct overload : Fs... { using Fs::operator()...; };

template <typename... Fs>
overload(Fs...) -> overload<Fs...>;

template <typename Variant, typename... Fs>
decltype(auto) match(Variant&& variant, Fs&&... fs)
{
  return std::visit(
      overload{std::forward<Fs>(fs)...},
      std::forward<Variant>(variant));
}

template <typename Variant, typename... Fs>
decltype(auto) match_partial(Variant&& variant, Fs&&... fs)
{
  return match(
      std::forward<Variant>(variant),
      std::forward<Fs>(fs)...,
      [](auto&&) {});
}
```

The overload helper exposes each callable's `operator()` as a single overload
set. `lib::match` passes that set to `std::visit`. `lib::match_partial` adds
a no-op fallback for alternatives the caller intentionally ignores.

```cpp
auto describe(const logging::logger& current) -> std::string
{
  return lib::match(current,
    [](const logging::console_logger&)  { return std::string{"console"}; },
    [](const logging::file_logger& f)   { return f.path.string(); },
    [](const logging::null_logger&)     { return std::string{"disabled"}; });
}
```

Each lambda binds to one alternative; the overload-set machinery picks the
right one for the active variant. Use `lib::match` for domain decisions that
should be revisited when a new alternative appears. Use `lib::match_partial`
for observation paths where ignored alternatives really mean "do nothing."

## See also

- `compile-time-correctness.md` -- the variant + match pattern composes with
  the `static constexpr` discriminator and parse-don't-validate ideas: the
  same visitor machinery dispatches on a domain variant whose
  alternatives carry their type tag as a static member.
- `architecture.md` -- functional core, imperative shell. `main` is the
  imperative shell where the variant gets configured; the inner layers reach
  the service through a free function and remain testable in isolation.
- `error-handling.md` -- errors are constructed by free functions that may
  themselves dispatch on a logging variant; the same pattern keeps the
  failure path and the diagnostic path on the same dispatch shape.

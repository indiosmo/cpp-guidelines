# C++ Design Principles

A guide to shaping modern C++ components around clear ownership, explicit
contracts, and local reasoning. The parent README defines the suite-wide
themes and conventions; this guide applies them to design.

## C++ standard

The guide assumes C++26. The table below lists fallbacks for the
features used in examples; a blank cell means the feature is already
available in that standard. Rows that read "no clean fallback" call out
patterns that simply do not translate.

| Feature                                      | C++23 fallback                                       | C++20 fallback                                                  |
|----------------------------------------------|------------------------------------------------------|-----------------------------------------------------------------|
| `std::expected<T, E>`                        |                                                      | project backport or `tl::expected<T, E>`                        |
| `std::unreachable()`                         |                                                      | compiler intrinsic such as `__builtin_unreachable()`            |
| `std::ranges::fold_left`                     |                                                      | `std::accumulate` over `begin()`/`end()`                        |
| `std::ranges::to<C>()`                       |                                                      | construct `C` from the view's `begin()`/`end()`                 |
| `std::stacktrace`                            |                                                      | header-only `cpptrace` or `boost::stacktrace`                   |
| `std::print` / `std::println`                |                                                      | `fmt::print` / `std::format` + `std::fputs`                     |
| `std::function_ref<R(Args...)>`              | header-only `tl::function_ref`                       | header-only `tl::function_ref`                                  |
| Contracts (`pre`, `post`, `contract_assert`) | `MY_ASSERT` / explicit `if`-and-return at entry/exit | `MY_ASSERT` / explicit `if`-and-return at entry/exit            |
| `std::start_lifetime_as<T>`                  |                                                      | `std::memcpy` into a `T` instance; no clean in-place fallback   |
| `std::generator<T>`                          |                                                      | hand-rolled coroutine traits or `cppcoro::generator`            |
| `std::mdspan`                                |                                                      | `Kokkos::mdspan` reference implementation                       |
| `std::flat_map` / `std::flat_set`            |                                                      | `boost::container::flat_map` / `flat_set`                       |

## The `lib::` namespace

The parent README introduces the `lib::` placeholder namespace used across the
suite. This guide uses two design-specific helpers often:

- `lib::strong_type<T, Tag>` is a strong type wrapper around a primitive,
  so two aliases over the same underlying type stay distinct. The `Tag`
  parameter is a phantom type used only for identity. Implementations
  exist in `boost::strong_typedef`, `NamedType`, and similar libraries.
- `lib::fixed_string<N>` is a non-allocating, fixed-capacity string with a
  compile-time maximum length -- the string analogue of
  `boost::container::static_vector`. `boost::static_string<N>` and the
  in-development `std::inplace_string<N>` proposal cover the same shape.

## Navigation

| File                              | Covers                                                                                                  |
|-----------------------------------|---------------------------------------------------------------------------------------------------------|
| `architecture.md`                 | Theory of the domain, domain separation with adapters, forward-only dependencies, functional core / imperative shell. |
| `compile-time-correctness.md`     | Strong typing, the per-domain `types` namespace, parse-don't-validate, plain-aggregate data structs, exhaustiveness checking, designated initializers. |
| `declarative-style.md`            | Decompose, work on simpler types, stage variables upfront, named predicates, ranges, lazy composition.  |
| `functional-programming.md`       | Pure functions and value semantics, sum-type matching with `lib::match`, higher-order functions, type-erased callables, capture lifetimes. |
| `templates.md`                    | Concepts for template constraints; inline `requires` for compile-time dispatch on capability.           |
| `preprocessor-macros.md`          | Boost.PP sub-header discipline, indirect three-token paste for unique temporaries, variadic dispatch via `__VA_OPT__`. |
| `error-handling.md`               | Result types, per-domain error codes, exceptions at boundaries, diagnostic context.                     |
| `invariants.md`                   | Exception safety, transactions, rollback with scope guards, idempotency for retried operations.         |
| `state-machines.md`               | When to model logic as an FSM, Boost.SML with embedded PlantUML, isolated transition tests.             |
| `pipelines.md`                    | Components as pipeline stages with `on_*` callback fields; wiring near `main` as a combinator (threading, tee, filter, deferred re-entrance); external SDK wrappers as event sources. |
| `runtime.md`                      | Threading as an edge effect: single-threaded component internals, posting work between threads via event-loop queues, lock-free primitives for small shared state. |
| `cross-cutting.md`                | Variant-based injectable globals (clock, logger, timer) with free-function dispatch via `lib::match`.   |
| `comments.md`                     | Guiding comments for non-obvious lines and blocks; comments as present-tense intent rather than history. |
| `performance.md`                  | Performance philosophy (clarity first, measure don't guess), hot-path allocations, flat vs node hash maps, type-erased callables. |

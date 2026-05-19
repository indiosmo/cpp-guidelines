# C++ Conventions Agent Context

Compact, always-loaded rules to pair with
[`cpp-design-principles-agent-context.md`](cpp-design-principles-agent-context.md).
Each section states the rule. Good/bad code pairs live in
[`cpp-conventions-examples.md`](cpp-conventions-examples.md); load that
file when shaping a concrete edit.

Examples in the companion file use `lib::result<T>` for the project's
`boost::leaf::result<T>` alias, `lib::strong_type<T, Tag>` for the local
strong-type helper, `lib::fixed_string<N>` for bounded hot-path text,
`lib::inplace_function<Sig, Capacity>` for fixed-capacity callbacks,
`lib::scope_exit` for rollback guards, and `lib::match` for exhaustive
variant visitation. Map the placeholders to the local vocabulary first.

## Layout

Use the project's component layout. The common shape is `src/<comp>/<comp>/`
for public headers, `src/<comp>/src/` for implementation, and
`test/<comp>/` for tests. Functional code lives in the component root.
Runtime wrappers live under `<comp>/runtime/`. Adapters that translate
between domains carry both domain names (`aor_fwll`, `aorfix_onixs_fix`).

## Tests

Drive the public domain surface with values; capture observable outputs
through callbacks or returned values. Do not spin runtime threads,
sockets, event loops, or SDK fixtures unless the test is explicitly
about that integration boundary.

Use factories, presets, and probes to keep setup out of the behavioral
story. Factories let a test override only the fields that matter. Probes
target narrow internal hygiene or branch setup that the public API
should not expose. For tests, sandboxes, and spikes, wire explicit noop
callbacks for irrelevant outputs; production wiring must assign real
callbacks.

When an expectation is non-obvious, cite its source in the test comment
or row label: a protocol rule, public spec, fixture, black-box scenario,
or bug report. Type-level contracts go in `static_assert` or
`STATIC_REQUIRE`, not runtime scaffolding.

## Debugging

Use assertions for programmer invariants that should be impossible in
correct code. Return structured errors for invalid input or
caller-visible failure conditions. If a fix adds a runtime check, decide
whether the stronger fix is a refined type, parse-at-boundary
conversion, boundary validation, rollback guard, or explicit error path.

For flaky tests, suspect timing and leaked state first. Replace sleeps
with predicate waits. Wrap global providers, singleton alternatives,
clocks, environment variables, and other process-wide state in RAII
guards so each test restores the previous state even after a `REQUIRE`
failure.

When the failure is deep in a call chain, add temporary logging before
the suspect call so the log captures the inputs that caused the failure.
For threaded or callback-heavy paths, capture enough context to identify
the request, branch, or test case that reached the bad state.

## Domain Types

Put `types.hpp` in every domain from the start. Define domain types
inside a nested `types` namespace. Keep the `types::` qualifier even
inside the domain; it makes ownership visible and avoids field/type name
collisions.

## Strong-Type Ergonomics

Use the strong type directly when the operation can stay in the strong
type. Same-type arithmetic preserves the strong type. Mixed expressions
may fall back to the underlying type when the local helper allows
implicit underlying references; re-wrapping a mixed result must be
explicit at the call site so domain mismatches stay reviewable. Use
`.get()` only at a real boundary that requires the primitive.

## Cross-Domain Strong-Type Construction

Do not write conversion helpers solely to move between two strong types
with the same safe representation. Construct the target strong type
directly. Use a named conversion helper only when the mapping is
semantic, lossy, fallible, or shape-changing (enum to wire integer,
parse to refined type, narrowing fixed string).

## Designated Initializers

Use designated initializers for aggregates. Use trailing commas on
multi-line initializers. Omit fields that already have defaults. Use
brace elision for non-scalar fields (strong types, containers); use `=`
for scalars.

## Namespace Aliases

Never put namespace aliases or `using` declarations in headers; they
leak into includers and make lookup depend on include order. In `.cpp`
and test files, use aliases only when a file repeatedly mixes domain
vocabularies; pick short, stable aliases already common in the project.
Local aliases inside a function for dense template or FSM namespaces are
fine when they improve readability.

## Const Placement

Use west const: `const T&`, not `T const&`.

## Comments

Use `/* ... */` for class, struct, header, and larger API documentation.
Use `//` for guiding comments inside function bodies.

Comment present intent, non-obvious behavior, and non-obvious ordering.
Do not narrate obvious code, describe old behavior, keep a trail of past
decisions, record a migration trail, or list what another layer handles.

Lean toward comments that summarize a block, even when the syntax is not
complex. Consecutive block comments should read like the function's
table of contents.

Comment non-obvious single lines: dense punctuation, designated
initializers across domains, chained calls, iterator invalidation,
lifetime contracts, performance-sensitive ordering, and protocol rules.
Skip comments for short, self-describing one-liners and repeatable local
idioms. A plain `lib::match` dispatcher is self-evident even if the
syntax looks busy; comment the non-obvious choice inside the visitor,
not the fact that variant dispatch is happening.

If a precondition matters, phrase it as the invariant the code relies
on, not as a chore the caller has been assigned.

## Error Types

Each domain owns `<domain>/error_code.hpp` (enum class + `std::error_code`
plumbing) and `<domain>/errors.hpp` (structured error payloads with
`code()` and `what()`). Generic errors are fine for generic
infrastructure failures such as bad config; domain failures carry
structured domain context.

## Result Flow

Inside a domain, compose fallible steps with `BOOST_LEAF_ASSIGN` and
`BOOST_LEAF_CHECK`. At public boundaries, consume `lib::result` with
`boost::leaf::try_handle_all` or `try_handle_some`, then return the
boundary's vocabulary. Allowed exception: a utility algorithm whose
public contract is "this fallible algorithm returns the project result
type."

## Error Handler Ordering

LEAF handlers are ordered. Put specific, intentional cases first and
catch-alls last; a catch-all placed first shadows every following
specific handler.

## Exhaustiveness

For `enum class`, switch without `default` so the compiler catches
missing enumerators; return a structured error after the switch for
corrupted or deserialized impossible values. For untyped boundary
values, use `default` because the compiler cannot help.

## Variants

Use `lib::match` for domain decisions. Add one arm per meaningful
alternative. Use `lib::match_partial` only for intentional subscriptions
where ignored alternatives are no-ops.

## Pipeline Callbacks

Stage callbacks are public wiring fields, usually `on_<event>`, stored
as `lib::inplace_function`. Production wiring assigns every
non-defaulted callback before it can fire. For tests, sandboxes, and
constructor-failure cleanup only, wire explicit noop callbacks for
irrelevant outputs. Do not hide missing production wiring with default
noops unless the callback is truly optional.

## Scope Guards

Use scope guards when partial mutation must be rolled back on any
failure path. Dismiss guards only after the whole operation commits.

## Runtime Posts

Post across threads. Capture moved messages by value. Keep closures
small enough for the project's inplace-function task storage. Do not
wait synchronously for a result from the target loop; model replies as
events posted back.

## State Machines

Put Boost.SML state machine vocabulary in a detail namespace. Use `ev_*`
event tags, `st_*` state tags, an `actions` struct of injected
callbacks, and a transition table. The owner stores actions before the
machine. For non-trivial machines, put a PlantUML
`/* @startuml ... @enduml */` block immediately above the transition
table and update it with every table change.

## Concepts And Requires

Use concepts for public template contracts. Use inline `requires` for
local capability dispatch.

## Hot-Path Storage

Use bounded storage on measured or explicitly constrained hot paths
(`lib::fixed_string<N>`, reserved vectors, `lib::inplace_function`).
Use node maps when references or pointers must survive inserts and
rehashes. Use flat maps for cache-friendly lookup tables where no
long-lived reference is kept.

## Result Pipeline Extraction

When a function has three or more sequential fallible steps, extract
named helpers and let the body read as the success path. Skip the
extraction for one or two obvious checks, or when the scaffolding would
create more error types than clarity.

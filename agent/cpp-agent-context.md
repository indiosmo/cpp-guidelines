# C++ Agent Context

On-demand reference companion to
[`cpp-agent-essentials.md`](cpp-agent-essentials.md). The essentials
file carries the high-leverage rules that shape code an agent would
otherwise write; this file holds the fuller treatment plus topics
demoted from the always-loaded path -- layout, strong-type
ergonomics, designated-initializer style, namespace alias and const
placement, error-handler ordering, exhaustiveness defaults, FSM
vocabulary, declarative style, cross-cutting services, and
forward-dependency framing. Load this file when an edit needs more
than the essentials, when a rule there feels under-specified, or when
the change touches one of the demoted topics. Concrete good/bad code
pairs live in
[`cpp-design-principles-examples.md`](cpp-design-principles-examples.md)
and [`cpp-conventions-examples.md`](cpp-conventions-examples.md);
load them when shaping a specific edit or reviewing a concrete shape.

Examples use `lib::` placeholders -- map them to the local project
vocabulary first. Common placeholders: `lib::result<T>` for the
project's `boost::leaf::result<T>` alias, `lib::strong_type<T, Tag>`
for the strong-type helper, `lib::fixed_string<N>` for bounded
hot-path text, `lib::inplace_function<Sig, Capacity>` for
fixed-capacity callbacks, `lib::scope_exit` for rollback guards, and
`lib::match` for exhaustive variant visitation.

Before writing code, search for the local abstraction that already
expresses the idea. Prefer project helpers over new utilities unless
the same missing shape recurs.

## Core Lens

Code is the working theory of the domain. Preserve that theory in the
names, types, dependency graph, and tests. When adding behavior, ask:

- Which domain owns this concept?
- Which type can carry the proof that this value is valid?
- Which boundary should translate this to another domain or external API?
- Which effect can move outward toward the runtime shell?
- Which invariant must remain true if a middle step fails?

If the answer is unclear, do a small local refactor that makes
ownership, inputs, outputs, and failure modes visible in the
signatures.

## Layout

Use the project's component layout. The common shape is
`src/<comp>/<comp>/` for public headers, `src/<comp>/src/` for
implementation, and `test/<comp>/` for tests. Functional code lives in
the component root. Runtime wrappers live under `<comp>/runtime/`.
Adapters that translate between domains carry both domain names
(`aor_fwll`, `aorfix_onixs_fix`).

## Tests

Tests are part of the design feedback loop. Each test encodes an
independent statement of intended behavior, derived from the domain,
contract, or specification. Do not derive the expected value from the
implementation under test; that only proves the code does what the
code does.

For bug fixes, write or tighten a test that fails for the observed
bug and passes for the intended behavior, then change production code
until the test passes. For new behavior, write the test first when
practical so the contract shapes the implementation.

Keep tests dense with signal: name a meaningful scenario, isolate the
behavior, and assert outcomes that fail for plausible bugs. Drive the
public domain surface with values; capture observable outputs through
callbacks or returned values. Component tests drive the functional
core directly; do not spin runtime threads, sockets, event loops, or
SDK fixtures unless the test is explicitly about that integration
boundary.

Use factories, presets, and probes to keep setup out of the
behavioral story. Factories let a test override only the fields that
matter. Probes target narrow internal hygiene or branch setup that
the public API should not expose. For tests, sandboxes, and spikes,
wire explicit noop callbacks for irrelevant outputs; production
wiring must assign real callbacks.

When an expectation is non-obvious, cite its source in the test
comment or row label: a protocol rule, public spec, fixture,
black-box scenario, or bug report. Type-level contracts belong in
compile-time tests (`static_assert`, `STATIC_REQUIRE`, concept
checks), not runtime scaffolding.

## Debugging

Do not patch symptoms. Reproduce the failure, read the whole error or
trace, and trace the bad value, state transition, or missing check
back to where it originated. If a fix attempt fails, treat that as
new evidence and return to investigation instead of stacking another
guess on top.

Form one explicit hypothesis at a time and test it with the smallest
useful experiment. Compare the broken path against a nearby working
path before inventing a new pattern; the difference is usually where
the bug lives.

Once the root cause is understood, write the regression test that
fails for that cause and passes for the intended behavior. Then fix
the source of the bug, not only the frame where it surfaced.
Afterward, ask what type, parser, boundary validation, assertion,
rollback guard, or error path would make the bug class harder to
reintroduce.

Use assertions for programmer invariants that should be impossible in
correct code. Return structured errors for invalid input or
caller-visible failure conditions. If a fix adds a runtime check,
decide whether the stronger fix is a refined type, parse-at-boundary
conversion, boundary validation, rollback guard, or explicit error
path.

For flaky tests, suspect timing and leaked state first. Replace
sleeps with predicate waits. Wrap global providers, singleton
alternatives, clocks, environment variables, and other process-wide
state in RAII guards so each test restores the previous state even
after a `REQUIRE` failure.

When the failure is deep in a call chain, add temporary logging
before the suspect call so the log captures the inputs that caused
the failure. For threaded or callback-heavy paths, capture enough
context to identify the request, branch, or test case that reached
the bad state.

## Domain Ownership

Each domain owns its types, messages, errors, and invariants. Sibling
domains do not share mutable vocabulary just because two fields share
a primitive representation. Adapters translate between neighboring
domains.

A construction such as `risk::types::order_id{order.price}` should
look wrong in review because the names and domains are wrong, not
because the code uses the wrong helper function.

Example manifestations: `order_routing` owns inbound requests,
`market_data` owns outbound messages, and `matching_engine` is the
composition point. Core domains and vendor wrappers stay separate;
adapter domains translate between them.

Put `types.hpp` in every domain from the start. Define domain types
inside a nested `types` namespace. Keep the `types::` qualifier even
inside the domain; it makes ownership visible and avoids field/type
name collisions.

## Forward Dependencies

Dependencies form a DAG. Headers, libraries, and call graphs read in
one direction. When two components need the same facility, extract a
third component both can depend on. Peers do not know each other
directly.

Avoid implicit contracts where one function leaves residue for
another to consume. Take what a function needs and return what it
produces.

## Types Carry Proof

Push correctness into the compiler. Wrap confusing primitives in
strong types, parse untyped input once at the boundary, use plain
aggregate data structs, designated initializers, `enum class`, and
exhaustive variant or enum handling.

Parse at system boundaries: wire bytes, JSON, CLI flags, config,
shared memory, database rows. Inside the domain, trust refined types.
Re-parse only when data re-enters from an untyped source.

## Strong-Type Ergonomics

Strong types stay ergonomic so they get used. Use the strong type
directly when the operation can stay in the strong type. Same-type
arithmetic preserves the strong type. Mixed expressions may fall back
to the underlying type when the local helper allows implicit
underlying references; re-wrapping a mixed result must be explicit at
the call site so domain mismatches stay reviewable. Use `.get()` only
at a real boundary that requires the primitive.

Do not write conversion helpers solely to move between two strong
types with the same safe representation. Construct the target strong
type directly. Use a named conversion helper only when the mapping is
semantic, lossy, fallible, or shape-changing (enum to wire integer,
parse to refined type, narrowing fixed string).

## Designated Initializers

Use designated initializers for aggregates. Use trailing commas on
multi-line initializers. Omit fields that already have defaults. Use
brace elision for non-scalar fields (strong types, containers); use
`=` for scalars.

## Functional Core, Imperative Shell

Domain code is synchronous logic over values. Threads, sockets,
timers, files, environment variables, clocks, dependency selection,
module wiring, and external SDK callbacks live at the edge.

This is the most important design pressure in the guide. The
functional component expresses domain rules; the shell decides where
inputs come from, which concrete services are installed, which thread
runs each consumer, and which module receives each callback. A
component that opens files, starts threads, reaches into a global SDK
session, constructs its own concrete dependency, or knows its
downstream consumer has pulled runtime composition into the domain.

Runtime wrappers compose the pure pieces near `main`. They load
config, parse wire bytes, install concrete services, create threads,
and connect module edges. Unit tests exercise the core without
spinning threads, sockets, event loops, vendor SDKs, or
dependency-injection containers.

## Pipelines

Pipeline stages expose inbound member functions and outbound callback
fields. The stage does not know who calls it or who consumes its
output. Wiring near `main` assigns the callbacks and is the only
place that sees the topology.

Stage callbacks are public wiring fields, usually `on_<event>`,
stored as `lib::inplace_function`. Production wiring assigns every
non-defaulted callback before it can fire. For tests, sandboxes, and
constructor-failure cleanup only, wire explicit noop callbacks for
irrelevant outputs. Do not hide missing production wiring with
default noops unless the callback is truly optional.

Put cross-cutting behavior at the wiring point: thread marshalling,
telemetry, filtering, test capture, and re-entrance deferral. A stage
does not grow a subscriber list or thread policy unless that is its
actual domain.

## Runtime And Threads

Threading is an edge effect. A component owns state on one thread;
other threads post work to that owner. Inner components carry no
atomics, mutexes, or condition variables by default.

Capture posted work by value unless the referenced lifetime is
obvious. Keep closures small enough for the project's
inplace-function task storage. Do not block the posting thread
waiting for a reply; send the reply as another posted event.

## Error Handling

Inside a domain, fallible helpers return the project result type
(`lib::result<T>`, usually a `boost::leaf::result<T>` alias). Compose
fallible steps with `BOOST_LEAF_ASSIGN` and `BOOST_LEAF_CHECK`. At a
domain or module boundary, consume the result with
`boost::leaf::try_handle_all` or `try_handle_some`, then return the
caller's vocabulary: `void`, `std::optional<T>`, or
`std::expected<T, E>`. Allowed exception: a utility algorithm whose
public contract is "this fallible algorithm returns the project
result type."

Top-level functions catch everything they produce. Registered
callbacks run on the caller's policy; dispatch them outside the
domain's `try_handle_all` when a callback exception should belong to
the caller.

Each domain owns `<domain>/error_code.hpp` (enum class +
`std::error_code` plumbing) and `<domain>/errors.hpp` (structured
error payloads with `code()` and `what()`). Generic errors are fine
for generic infrastructure failures such as bad config; domain
failures carry structured domain context.

LEAF handlers are ordered. Put specific, intentional cases first and
catch-alls last; a catch-all placed first shadows every following
specific handler.

For `enum class`, switch without `default` so the compiler catches
missing enumerators; return a structured error after the switch for
corrupted or deserialized impossible values. For untyped boundary
values, use `default` because the compiler cannot help.

When a function has three or more sequential fallible steps, extract
named helpers and let the body read as the success path. Skip the
extraction for one or two obvious checks, or when the scaffolding
would create more error types than clarity.

## Invariants And Rollback

Mutating methods that maintain non-trivial invariants target the
strong exception guarantee: the operation either fully applies or has
no effect. Prefer commit-at-end when possible. Use scope guards when
a later fallible step must observe an earlier mutation; dismiss
guards only after the whole operation commits.

Idempotent operations record exactly what they applied so retries and
rollback do not recompute from drifted state.

## Declarative Style

Decompose, work on simpler inputs, stage derived values upfront, name
predicates, and use algorithms, ranges, and variant matching. The
success path reads as a sequence of domain steps, not as plumbing.

## Variants, Concepts, And Templates

Use `lib::match` for domain decisions over `std::variant`. Add one
arm per meaningful alternative; missing alternatives fail to compile.
Use `lib::match_partial` only for intentional subscriptions where
ignored alternatives are no-ops. Prefer capability dispatch
(`if constexpr (requires { alt.request_id; })`) to type-name
dispatch.

Use concepts for public template contracts. Use inline `requires` for
local capability dispatch. Use dependent-false helpers when an
unsupported template instantiation should fail only after selection.

## State Machines

Reach for an FSM when behavior is a graph of legal moves: protocol
lifecycle, connection state, retry paths, entry and exit actions, and
correlated booleans. Use a small function or `std::variant` for
one-bit or linear flows.

Put Boost.SML state machine vocabulary in a detail namespace. Use
`ev_*` event tags, `st_*` state tags, an `actions` struct of injected
callbacks, and a transition table. The owner stores actions before
the machine. For non-trivial machines, put a PlantUML
`/* @startuml ... @enduml */` block immediately above the transition
table and update it with every table change.

## Cross-Cutting Services

For genuine cross-cutting services such as clock, logger, timer, and
metrics, prefer a variant-backed global configured once in `main`
plus free functions that dispatch through `lib::match`. Do not thread
unused service references through every constructor.

Reserve this pattern for services that truly cut across layers.
Ordinary domain dependencies belong in signatures or adapters.

## Performance Discipline

Start with clear code and a known requirement. Optimize measured hot
paths, not hunches. Keep allocation, locality, and algorithmic
complexity in view, but do not contort cold code.

Use bounded storage on measured or explicitly constrained hot paths:
bounded text via `lib::fixed_string<N>`; bounded vectors via
`boost::container::static_vector<T, N>`; growing vectors reserved
once at startup; stored callbacks via `lib::inplace_function`. Use
node maps when references or pointers must survive inserts and
rehashes. Use flat maps for cache-friendly lookup tables where no
long-lived reference is kept.

Document a non-obvious performance trade-off with the requirement or
benchmark that justifies it. Otherwise write the obvious version.

## Comments

Use comments as local statements of intent. Use `/* ... */` for
class, struct, header, and larger API documentation. Use `//` for
guiding comments inside function bodies.

Comment present intent, non-obvious behavior, and non-obvious
ordering. For non-obvious lines or blocks, write a short English
sentence before the code so the reader can skim the function's flow
before parsing C++ syntax. Good comments also give reviewers a claim
to compare against the implementation.

Lean toward commenting blocks even when the syntax is simple: a
loop, conditional, lambda body, or related group of statements is
faster to read when preceded by one sentence naming the step.
Consecutive block comments should read like the function's table of
contents.

Comment non-obvious single lines: dense punctuation, designated
initializers across domains, chained calls, iterator invalidation,
lifetime contracts, performance-sensitive ordering, and protocol
rules. Skip comments for short, self-describing one-liners and
repeatable local idioms. A plain `lib::match` dispatcher is
self-evident even if the syntax looks busy; comment the non-obvious
choice inside the visitor, not the fact that variant dispatch is
happening.

Do not narrate obvious code, describe old behavior, keep a trail of
past decisions, record a migration trail, or list what another layer
handles. Comments describe the present code and the present reason
for its shape. Git history records prior designs; ADRs hold enduring
decision rationale.

If a precondition matters, phrase it as the invariant the code
relies on, not as a chore the caller has been assigned.

## Namespace Aliases

Never put namespace aliases or `using` declarations in headers; they
leak into includers and make lookup depend on include order. In
`.cpp` and test files, use aliases only when a file repeatedly mixes
domain vocabularies; pick short, stable aliases already common in the
project. Local aliases inside a function for dense template or FSM
namespaces are fine when they improve readability.

## Const Placement

Use west const: `const T&`, not `T const&`.

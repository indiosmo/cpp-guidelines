# C++ Agent Essentials

Compact, always-loaded cheat sheet for agents working in modern C++
codebases. Keeps only the rules that change the shape of code an agent
would otherwise write. The full reference -- strong-type ergonomics
minutiae, designated-initializer style, namespace alias and const
placement, error-handler ordering, exhaustiveness defaults, FSM
vocabulary, concepts vs `requires` -- lives in the on-demand companion
[`cpp-agent-context.md`](cpp-agent-context.md). Concrete good/bad code
pairs live in
[`cpp-design-principles-examples.md`](cpp-design-principles-examples.md)
and [`cpp-conventions-examples.md`](cpp-conventions-examples.md). Load
any of these when shaping a specific edit.

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

## Domain Ownership

Each domain owns its types, messages, errors, and invariants. Sibling
domains do not share mutable vocabulary just because two fields share
a primitive representation. Adapters translate between neighboring
domains.

A construction such as `risk::types::order_id{order.price}` should
look wrong in review because the names and domains are wrong, not
because the code uses the wrong helper function.

Put `types.hpp` in every domain. Define domain types inside a nested
`types` namespace and keep the `types::` qualifier even inside the
domain; it makes ownership visible and avoids field/type name
collisions.

## Types Carry Proof

Push correctness into the compiler. Wrap confusing primitives in
strong types, use plain aggregate data structs, designated
initializers, `enum class`, and exhaustive variant or enum handling.

Parse at system boundaries: wire bytes, JSON, CLI flags, config,
shared memory, database rows. Inside the domain, trust refined types.
Re-parse only when data re-enters from an untyped source.

## Functional Core, Imperative Shell

Domain code is synchronous logic over values. Threads, sockets,
timers, files, environment variables, clocks, dependency selection,
module wiring, and external SDK callbacks live at the edge.

This is the most important design pressure in the guide. Runtime
wrappers compose the pure pieces near `main`: they load config, parse
wire bytes, install concrete services, create threads, and connect
module edges. A component that opens files, starts threads, reaches
into a global SDK session, constructs its own concrete dependency, or
knows its downstream consumer has pulled runtime composition into the
domain.

Unit tests exercise the core without spinning threads, sockets, event
loops, vendor SDKs, or dependency-injection containers.

## Pipelines

Pipeline stages expose inbound member functions and outbound callback
fields stored as `lib::inplace_function`, usually `on_<event>`. The
stage does not know who calls it or who consumes its output. Wiring
near `main` assigns the callbacks and is the only place that sees the
topology.

Production wiring assigns every non-defaulted callback before it can
fire. For tests, sandboxes, and constructor-failure cleanup only, wire
explicit noop callbacks for irrelevant outputs. Cross-cutting behavior
-- thread marshalling, telemetry, filtering, test capture,
re-entrance deferral -- belongs at the wiring point, not inside the
stage.

## Threading

Threading is an edge effect. A component owns state on one thread;
other threads post work to that owner. Inner components carry no
atomics, mutexes, or condition variables by default.

Capture posted work by value unless the referenced lifetime is
obvious. Do not block the posting thread waiting for a reply; send the
reply as another posted event.

## Error Handling

Inside a domain, fallible helpers return `lib::result<T>` (the project
`boost::leaf::result<T>` alias). Compose them with `BOOST_LEAF_ASSIGN`
and `BOOST_LEAF_CHECK`. At a domain or module boundary, consume the
result with `boost::leaf::try_handle_all` or `try_handle_some`, then
return the caller's vocabulary: `void`, `std::optional<T>`, or
`std::expected<T, E>`.

Each domain owns `<domain>/error_code.hpp` (enum class +
`std::error_code` plumbing) and `<domain>/errors.hpp` (structured
error payloads with `code()` and `what()`). Generic errors are fine
for generic infrastructure failures; domain failures carry structured
domain context.

When a mutating method maintains a non-trivial invariant, target the
strong exception guarantee: prefer commit-at-end, and use scope guards
when a later fallible step must observe an earlier mutation. Dismiss
guards only after the whole operation commits.

## Variants

Use `lib::match` for domain decisions over `std::variant`. Add one arm
per meaningful alternative; missing alternatives fail to compile. Use
`lib::match_partial` only when ignored alternatives are intentionally
no-ops. Prefer capability dispatch
(`if constexpr (requires { alt.request_id; })`) to type-name dispatch.

## Hot-Path Storage

Start with clear code and a known requirement; optimize measured hot
paths, not hunches. On measured or explicitly constrained hot paths,
use bounded storage:

- bounded text via `lib::fixed_string<N>`;
- bounded vectors via `boost::container::static_vector<T, N>`;
- growing vectors reserved once at startup;
- stored callbacks via `lib::inplace_function`.

Use node maps when references or pointers must survive inserts and
rehashes; flat maps for cache-friendly lookup tables where no
long-lived reference is kept.

## Tests

Each test encodes an independent statement of intended behavior,
derived from the domain, contract, or specification. Do not derive the
expected value from the implementation under test; that only proves
the code does what the code does.

Drive the public domain surface with values; capture observable
outputs through callbacks or returned values. Component tests do not
spin runtime threads, sockets, event loops, or SDK fixtures unless the
test is explicitly about that integration boundary.

For bug fixes, write or tighten a test that fails for the observed bug
and passes for the intended behavior, then change production code
until the test passes. For new behavior, write the test first when
practical so the contract shapes the implementation.

## Debugging

Do not patch symptoms. Reproduce the failure, read the whole error or
trace, and trace the bad value, state transition, or missing check
back to where it originated. Form one explicit hypothesis at a time
and test it with the smallest useful experiment. Compare the broken
path against a nearby working path; the difference is usually where
the bug lives.

Once the root cause is understood, write the regression test that
fails for that cause and passes for the intended behavior. Then fix
the source of the bug, not only the frame where it surfaced.
Afterward, ask what type, parser, boundary validation, assertion,
rollback guard, or error path would make the bug class harder to
reintroduce.

## Comments

Comment present intent, non-obvious behavior, and non-obvious
ordering. Use `/* ... */` for class, struct, and header docs; `//` for
guiding comments inside function bodies. Lean toward commenting blocks
even when the syntax is simple: one sentence naming the step. A plain
`lib::match` dispatcher is self-evident; comment the non-obvious
choice inside the visitor, not the fact that variant dispatch is
happening.

Do not narrate obvious code, describe old behavior, keep a trail of
past decisions, record a migration trail, or list what another layer
handles. Comments describe the present code and the present reason
for its shape; git history records prior designs and ADRs hold
enduring decision rationale.

If a precondition matters, phrase it as the invariant the code relies
on, not as a chore the caller has been assigned.

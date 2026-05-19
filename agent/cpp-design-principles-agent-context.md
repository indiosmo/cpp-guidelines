# C++ Design Principles Agent Context

Compact, always-loaded context for agents working in modern C++ codebases.
This file distills the full `cpp-design-principles/` guide into the rules
most likely to change code shape. Concrete good/bad code pairs live in
[`cpp-design-principles-examples.md`](cpp-design-principles-examples.md);
load that file when designing or reviewing concrete shapes. Examples use
`lib::` placeholders -- map them to the local project vocabulary first.

Before writing code, search for the local abstraction that already
expresses the idea. Prefer project helpers over new utilities unless the
same missing shape recurs.

## Core Lens

Code is the working theory of the domain. Preserve that theory in the
names, types, dependency graph, and tests. When adding behavior, ask:

- Which domain owns this concept?
- Which type can carry the proof that this value is valid?
- Which boundary should translate this to another domain or external API?
- Which effect can move outward toward the runtime shell?
- Which invariant must remain true if a middle step fails?

If the answer is unclear, do a small local refactor that makes ownership,
inputs, outputs, and failure modes visible in the signatures.

## Testing Intent

Tests are part of the design feedback loop. Each test encodes an
independent statement of intended behavior, derived from the domain,
contract, or specification. Do not derive the expected value from the
implementation under test; that only proves the code does what the code
does.

For bug fixes, write or tighten a test that fails for the observed bug
and passes for the intended behavior, then change production code until
the test passes. For new behavior, write the test first when practical
so the contract shapes the implementation.

Keep tests dense with signal: name a meaningful scenario, isolate the
behavior, and assert outcomes that fail for plausible bugs. Type-level
contracts belong in compile-time tests (`static_assert`,
`STATIC_REQUIRE`, concept checks). Component tests drive the functional
core directly; avoid threads, sockets, event loops, and vendor SDKs
unless the test targets that integration boundary.

## Debugging Discipline

Do not patch symptoms. Reproduce the failure, read the whole error or
trace, and trace the bad value, state transition, or missing check back
to where it originated. If a fix attempt fails, treat that as new
evidence and return to investigation instead of stacking another guess
on top.

Form one explicit hypothesis at a time and test it with the smallest
useful experiment. Compare the broken path against a nearby working path
before inventing a new pattern; the difference is usually where the bug
lives.

Once the root cause is understood, write the regression test that fails
for that cause and passes for the intended behavior. Then fix the source
of the bug, not only the frame where it surfaced. Afterward, ask what
type, parser, boundary validation, assertion, rollback guard, or error
path would make the bug class harder to reintroduce.

## Domain Ownership

Each domain owns its types, messages, errors, and invariants. Sibling
domains do not share mutable vocabulary just because two fields share a
primitive representation. Adapters translate between neighboring domains.

A construction such as `risk::types::order_id{order.price}` should look
wrong in review because the names and domains are wrong, not because the
code uses the wrong helper function. The mechanics of cross-domain
construction belong in the conventions file.

Example manifestations: `order_routing` owns inbound requests,
`market_data` owns outbound messages, and `matching_engine` is the
composition point. Core domains and vendor wrappers stay separate;
adapter domains translate between them.

## Forward Dependencies

Dependencies form a DAG. Headers, libraries, and call graphs read in one
direction. When two components need the same facility, extract a third
component both can depend on. Peers do not know each other directly.

Avoid implicit contracts where one function leaves residue for another
to consume. Take what a function needs and return what it produces.

## Types Carry Proof

Push correctness into the compiler. Wrap confusing primitives in strong
types, parse untyped input once at the boundary, use plain aggregate
data structs, designated initializers, `enum class`, and exhaustive
variant or enum handling. Strong types stay ergonomic: same-type
operations preserve the strong type, and mixed-domain rewrapping is
explicit at the call site rather than hidden behind generic conversion
plumbing.

Parse at system boundaries: wire bytes, JSON, CLI flags, config, shared
memory, database rows. Inside the domain, trust refined types. Re-parse
only when data re-enters from an untyped source.

## Functional Core, Imperative Shell

Domain code is synchronous logic over values. Threads, sockets, timers,
files, environment variables, clocks, dependency selection, module
wiring, and external SDK callbacks live at the edge.

This is the most important design pressure in the guide. The functional
component expresses domain rules; the shell decides where inputs come
from, which concrete services are installed, which thread runs each
consumer, and which module receives each callback. A component that
opens files, starts threads, reaches into a global SDK session,
constructs its own concrete dependency, or knows its downstream consumer
has pulled runtime composition into the domain.

Runtime wrappers compose the pure pieces near `main`. They load config,
parse wire bytes, install concrete services, create threads, and connect
module edges. Unit tests exercise the core without spinning threads,
sockets, event loops, vendor SDKs, or dependency-injection containers.

## Pipelines

Pipeline stages expose inbound member functions and outbound callback
fields. The stage does not know who calls it or who consumes its output.
Wiring near `main` assigns the callbacks and is the only place that
sees the topology.

Put cross-cutting behavior at the wiring point: thread marshalling,
telemetry, filtering, test capture, and re-entrance deferral. A stage
does not grow a subscriber list or thread policy unless that is its
actual domain.

## Runtime And Threads

Threading is an edge effect. A component owns state on one thread; other
threads post work to that owner. Inner components carry no atomics,
mutexes, or condition variables by default.

Capture posted work by value unless the referenced lifetime is obvious.
Do not block the posting thread waiting for a reply; send the reply as
another posted event.

## Error Handling

Inside a domain, fallible helpers return the project result type
(`lib::result<T>`, usually a `boost::leaf::result<T>` alias). Domain
errors live with the producer in `error_code.hpp` and `errors.hpp`. At a
domain or module boundary, consume the result and return the caller's
vocabulary: `void`, `std::optional<T>`, or `std::expected<T, E>`.

Top-level functions catch everything they produce. Registered callbacks
run on the caller's policy; dispatch them outside the domain's
`try_handle_all` when a callback exception should belong to the caller.

## Invariants And Rollback

Mutating methods that maintain non-trivial invariants target the strong
exception guarantee: the operation either fully applies or has no
effect. Prefer commit-at-end when possible. Use scope guards when a
later fallible step must observe an earlier mutation.

Idempotent operations record exactly what they applied so retries and
rollback do not recompute from drifted state.

## Declarative Style

Decompose, work on simpler inputs, stage derived values upfront, name
predicates, and use algorithms, ranges, and variant matching. The
success path reads as a sequence of domain steps, not as plumbing.

For three or more sequential fallible steps, prefer named
`lib::result`-returning helpers composed with `BOOST_LEAF_ASSIGN` and
`BOOST_LEAF_CHECK` over a long inline `if (!ok)` cascade.

## Guiding Comments

Use comments as local statements of intent. For non-obvious lines or
blocks, write a short English sentence before the code so the reader
can skim the function's flow before parsing C++ syntax. Good comments
also give reviewers a claim to compare against the implementation.

Lean toward commenting blocks even when the syntax is simple: a loop,
conditional, lambda body, or related group of statements is faster to
read when preceded by one sentence naming the step. Do not comment
repeatable local idioms (a plain `lib::match` dispatcher) just because
the syntax looks busy.

Do not narrate obvious code, keep a trail of past decisions in comments,
or list what the code does not do. Comments describe the present code
and the present reason for its shape. Git history records prior designs;
ADRs hold enduring decision rationale. If a precondition matters, state
it as a present-tense invariant the code relies on.

## Variants, Concepts, And Templates

Use `lib::match` for domain decisions over `std::variant`; missing
alternatives fail to compile. Use `lib::match_partial` only when ignored
alternatives are intentionally irrelevant. Prefer capability dispatch
(`if constexpr (requires { alt.request_id; })`) to type-name dispatch.

Use concepts for reusable template contracts. Use dependent-false
helpers when an unsupported template instantiation should fail only
after selection.

## State Machines

Reach for an FSM when behavior is a graph of legal moves: protocol
lifecycle, connection state, retry paths, entry and exit actions, and
correlated booleans. Use a small function or `std::variant` for one-bit
or linear flows.

Keep side effects in an injected `actions` object. Put a PlantUML
comment beside the transition table when the graph is large enough to
review visually, and keep diagram and table in lockstep.

## Cross-Cutting Services

For genuine cross-cutting services such as clock, logger, timer, and
metrics, prefer a variant-backed global configured once in `main` plus
free functions that dispatch through `lib::match`. Do not thread unused
service references through every constructor.

Reserve this pattern for services that truly cut across layers. Ordinary
domain dependencies belong in signatures or adapters.

## Performance Discipline

Start with clear code and a known requirement. Optimize measured hot
paths, not hunches. Keep allocation, locality, and algorithmic
complexity in view, but do not contort cold code.

Hot-path swaps: bounded text via `lib::fixed_string<N>`; bounded vectors
via `boost::container::static_vector<T, N>`; growing vectors reserved
once at startup; stored callbacks via `lib::inplace_function`; flat maps
for locality, node maps when reference stability is load-bearing.

Document a non-obvious performance trade-off with the requirement or
benchmark that justifies it. Otherwise write the obvious version.

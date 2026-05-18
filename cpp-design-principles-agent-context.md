# C++ Design Principles Agent Context

Use this file as compact, always-loaded context for agents working in modern
C++ codebases. It distills the full `cpp-design-principles/` guide into the
rules most likely to change code shape. The examples use `lib::` placeholders;
map them to the local project vocabulary first (`kraken::`, `mil::`, or
another in-house namespace).

Before writing code, search for the local abstraction that already expresses
the idea. Prefer project helpers over new utilities unless the same missing
shape appears repeatedly.

## Core Lens

Code is the working theory of the domain. Preserve that theory in the names,
types, dependency graph, and tests. When adding behavior, ask:

- Which domain owns this concept?
- Which type can carry the proof that this value is valid?
- Which boundary should translate this to another domain or external API?
- Which effect can move outward toward the runtime shell?
- Which invariant must remain true if a middle step fails?

If the answer is unclear, do a small local refactor that makes ownership,
inputs, outputs, and failure modes visible in the signatures.

## Testing Intent

Tests are part of the design feedback loop. A test should encode an
independent statement of intended behavior, derived from the domain, contract,
or relevant specification. Do not derive the expected value from the
implementation under test; that only proves the code does what the code does.

When fixing a bug, first write or tighten a test that fails for the observed
bug and passes for the intended behavior. Then change the production code until
the test passes. When adding behavior, write the test first when practical so
the expected contract guides the implementation instead of echoing it.

Keep tests high in information density. Each case should name a meaningful
scenario, isolate the behavior being specified, and assert outcomes that would
fail for plausible bugs. Prefer a small set of contract-rich cases over a large
suite of mechanical cases that nobody can read or trust.

Use the right source for the expectation. Protocol and workflow tests should
cite the spec, fixture, or bug report that defines the observable behavior.
Type-level contracts belong in compile-time tests (`static_assert`,
`STATIC_REQUIRE`, concept checks) so unsupported constructions and missing
variant arms fail during compilation.

Keep component tests on the functional core. Drive the public domain surface
with values, capture emitted callbacks, and avoid threads, sockets, event
loops, vendor SDKs, and runtime wiring unless the behavior under test is
integration behavior.

## Debugging Discipline

Do not patch symptoms. Reproduce the failure, read the whole error or trace,
and trace the bad value, state transition, or missing check back to where it
originated. If a fix attempt fails, treat that as new evidence and return to
investigation instead of stacking another guess on top.

Form one explicit hypothesis at a time and test it with the smallest useful
experiment. Compare the broken path against a nearby working path before
inventing a new pattern; the difference is usually where the bug lives.

Once the root cause is understood, write the regression test that fails for
that cause and passes for the intended behavior. Then fix the source of the
bug, not only the frame where it surfaced. Afterward, ask what type, parser,
boundary validation, assertion, rollback guard, or error path would make the
bug class harder to reintroduce.

## Domain Ownership

Each domain owns its types, messages, errors, and invariants. Sibling domains
do not share mutable vocabulary just because two fields happen to have the
same primitive representation. Adapters translate between neighboring domains.

Good - same safe representation, no helper:

```cpp
namespace routing::types {
using order_id = lib::strong_type<lib::fixed_string<36>, struct OrderIdTag>;
}

namespace risk::types {
using order_id = lib::strong_type<lib::fixed_string<36>, struct OrderIdTag>;
}

void submit_to_risk(routing::types::order_id routing_id)
{
  auto risk_id = risk::types::order_id{routing_id};
  send_to_risk(risk_id);
}
```

Good - real conversion, helper earns its name:

```cpp
namespace routing::types {
enum class side : std::uint8_t { buy, sell };
}

namespace fix::values {
inline constexpr char buy = '1';
inline constexpr char sell = '2';
}

auto side_to_fix_char(routing::types::side side) -> lib::result<char>
{
  switch (side) {
    case routing::types::side::buy: return fix::values::buy;
    case routing::types::side::sell: return fix::values::sell;
  }

  return lib::make_leaf_error(routing::errors::invalid_side{.side = side});
}
```

Bad - helper adds no domain knowledge:

```cpp
auto to_risk_order_id(routing::types::order_id id) -> risk::types::order_id
{
  return risk::types::order_id{id};
}
```

Do not add conversion helpers just to move between strong types with the same
safe representation. Direct construction keeps the logic visible without
littering code with type machinery. A construction such as
`risk::types::order_id{order.price}` should look wrong in review because the
names and domains are wrong.

Use a named conversion helper when the mapping is semantic, lossy, fallible, or
shape-changing: enum to wire integer, enum to string, string to parsed enum, a
larger fixed string into a smaller one, or any conversion that can reject.

Assessment-style manifestation: `order_routing` owns inbound requests,
`market_data` owns outbound messages, and `matching_engine` is the composition
point. Abacus-style manifestation: `aor`, `aorfix`, and vendor wrappers stay
separate; adapter domains such as `aorfix_onixs_fix` and `aor_fwll` translate.

## Forward Dependencies

Dependencies form a DAG. Headers, libraries, and call graphs should read in
one direction. When two components need the same facility, extract a third
component both can depend on. Do not make peers know each other directly.

Good:

```cpp
auto untie(const table& data) -> column;
auto rank(const table& data, const column& tiebreaker) -> table;

auto score(const table& data) -> table
{
  return rank(data, untie(data));
}
```

Bad:

```cpp
struct working_set {
  table data;
  std::optional<column> tiebreaker; // set by untie(), consumed by rank()
};

void untie(working_set&);
void rank(working_set&); // only valid after untie()
```

Avoid implicit contracts where one function leaves residue for another to
consume. Take what a function needs and return what it produces.

## Types Carry Proof

Push correctness into the compiler. Wrap confusing primitives in strong types,
parse untyped input once at the boundary, use plain aggregate data structs,
designated initializers, `enum class`, and exhaustive variant/enum handling.
Strong types should still be ergonomic: same-type operations preserve the
strong type, and mixed-domain rewrapping is explicit at the call site rather
than hidden behind generic conversion plumbing.

Good:

```cpp
auto parse_symbol(std::string_view raw) -> lib::result<types::symbol>;

void place_order(types::account account, types::symbol symbol, types::quantity qty);
```

Bad:

```cpp
auto validate_symbol(std::string_view raw) -> bool;

void place_order(std::string_view account, std::string_view symbol, int qty);
// Every downstream caller must remember which strings were validated.
```

Parse at system boundaries: wire bytes, JSON, CLI flags, config, shared memory,
database rows. Once inside the domain, trust refined types. Re-parse only when
data re-enters from an untyped source.

## Functional Core, Imperative Shell

Domain code is synchronous logic over values. Threads, sockets, timers, files,
environment variables, clocks, dependency selection, module wiring, and
external SDK callbacks live at the edge.

This is the most important design pressure in the guide: push effects to the
edge. The functional component should express domain rules; the shell decides
where inputs come from, which concrete services are installed, which thread
runs each consumer, and which module receives each callback. A component that
opens files, starts threads, reaches into a global SDK session, constructs its
own concrete dependency, or knows its downstream consumer has pulled runtime
composition into the domain.

Good:

```cpp
auto apply_fill(order_state order, fill fill) -> order_state;
```

Bad:

```cpp
void apply_fill(order_state& order, fill fill)
{
  global_logger.info("fill {}", fill.id);
  background_executor.post([&] { persist(order); });
}
```

Runtime wrappers compose the pure pieces near `main`. They load config, parse
wire bytes, install concrete services, create threads, and connect module
edges. Unit tests should be able to exercise the core without spinning threads,
sockets, event loops, vendor SDKs, or dependency-injection containers.

Good:

```cpp
class risk_check {
public:
  auto check(const order& order, const limits& limits) -> lib::result<decision>;
};

// Near main: choose the provider, fetch the limits, call the pure component.
auto limits = limits_provider.load(account);
BOOST_LEAF_ASSIGN(auto decision, risk.check(order, limits));
```

Bad:

```cpp
class risk_check {
public:
  auto check(const order& order) -> lib::result<decision>
  {
    auto limits = database_limits_provider{config_path_}.load(order.account);
    return check_against_limits(order, limits);
  }
};
```

The bad version hides I/O, configuration, dependency construction, and failure
policy inside the domain component. The good version keeps those choices at the
runtime boundary.

## Pipelines

Pipeline stages expose inbound member functions and outbound callback fields.
The stage does not know who calls it or who consumes its output. Wiring near
`main` assigns the callbacks and is the only place that sees the topology.

Good:

```cpp
class order_router {
public:
  void send(new_order&& request);

  lib::inplace_function<void(routed_order&&)> on_routed;
  lib::inplace_function<void(rejection&&)> on_rejected;
};

router.on_routed = [&session](routed_order&& order) {
  session.send(std::move(order));
};
```

Put cross-cutting behavior at the wiring point: thread marshalling, telemetry,
filtering, test capture, and re-entrance deferral. A stage should not grow a
subscriber list or thread policy unless that is its actual domain.

## Runtime And Threads

Threading is an edge effect. A component owns state on one thread; other
threads post work to that owner. Inner components should not contain atomics,
mutexes, or condition variables by default.

Good:

```cpp
consumer_loop.post([&consumer, event = std::move(event)]() mutable {
  consumer.send(std::move(event));
});
```

Bad:

```cpp
// A direct cross-thread call into unsynchronized component state.
consumer.send(std::move(event));
```

Capture posted work by value unless the referenced lifetime is obvious. Do not
block the posting thread waiting for a reply; send a reply as another posted
event.

## Error Handling

Inside a domain, fallible helpers return the project result type
(`lib::result<T>`, usually a `boost::leaf::result<T>` alias). Domain errors
live with the producer in `error_code.hpp` and `errors.hpp`. At a domain or
module boundary, consume the result and return the caller's vocabulary:
`void`, `std::optional<T>`, or `std::expected<T, E>`.

Good:

```cpp
lib::result<void> process()
{
  BOOST_LEAF_ASSIGN(const auto& order, orders_.find(id));
  BOOST_LEAF_CHECK(validate(order));
  return {};
}

void handle(request&& req)
{
  boost::leaf::try_handle_all(
    [&]() -> lib::result<void> {
      BOOST_LEAF_CHECK(process(req));
      return {};
    },
    domain::make_error_handlers(req));
}
```

Bad:

```cpp
bool process(std::string* error_text); // untyped, easy to ignore, hard to route
```

Top-level functions catch everything they produce. Registered callbacks run on
the caller's policy; dispatch them outside the domain's `try_handle_all` when a
callback exception should belong to the caller.

## Invariants And Rollback

Mutating methods that maintain non-trivial invariants should target the strong
exception guarantee: the operation either fully applies or has no effect.

Prefer commit-at-end when possible:

```cpp
auto add(order o) -> lib::result<void>
{
  BOOST_LEAF_CHECK(validate(o));
  auto next = orders_;
  next.emplace(o.id, std::move(o));
  orders_ = std::move(next);
  return {};
}
```

Use scope guards when a later fallible step must observe an earlier mutation:

```cpp
orders_.emplace(order.id, order_state);
auto undo_order = lib::scope_exit([&] { orders_.erase(order.id); });

BOOST_LEAF_CHECK(limits_.reserve(order));

undo_order.dismiss();
```

Idempotent operations record exactly what they applied so retries and rollback
do not recompute from drifted state.

## Declarative Style

Decompose, work on simpler inputs, stage derived values upfront, name
predicates, and use algorithms, ranges, and variant matching. The success path
should read as a sequence of domain steps, not as plumbing.

Good:

```cpp
const bool has_residual = order.remaining_quantity > 0;
const bool is_market = order.limit_price == 0;
const bool should_rest = has_residual && !is_market;

if (should_rest) {
  rest(order);
}
```

Bad:

```cpp
if (order.remaining_quantity > 0 && !(order.limit_price == 0)) {
  rest(order);
}
```

For three or more sequential fallible steps, prefer named
`lib::result`-returning helpers composed with `BOOST_LEAF_ASSIGN` /
`BOOST_LEAF_CHECK` over a long inline `if (!ok) { log; return; }` cascade.

## Variants, Concepts, And Templates

Use `lib::match` for domain decisions over `std::variant`; missing alternatives
should fail to compile. Use `lib::match_partial` only when ignored alternatives
are intentionally irrelevant.

Prefer capability dispatch to type-name dispatch:

```cpp
return lib::match(event, [](const auto& alt) -> std::optional<types::request_id> {
  if constexpr (requires { alt.request_id; }) {
    return alt.request_id;
  }
  return std::nullopt;
});
```

Use concepts for reusable template contracts. Use dependent-false helpers when
an unsupported template instantiation should fail only after selection.

## State Machines

Reach for an FSM when behavior is a graph of legal moves: protocol lifecycle,
connection state, retry paths, entry/exit actions, and correlated booleans.
Use a small function or `std::variant` for one-bit or linear flows.

Good FSM shape:

```cpp
struct actions {
  lib::inplace_function<void()> async_connect;
  lib::inplace_function<void()> async_close;
};

struct transitions {
  auto operator()() const noexcept {
    namespace sml = boost::sml;
    return sml::make_transition_table(
      *sml::state<st_closed> + sml::event<ev_connect> = sml::state<st_connecting>,
      sml::state<st_connecting> + sml::on_entry<sml::_> /
        [](actions& a) { a.async_connect(); });
  }
};
```

Keep side effects in the injected `actions` object. Put a PlantUML comment
beside the transition table when the graph is large enough to review visually,
and keep diagram and table in lockstep.

## Cross-Cutting Services

For genuine cross-cutting services such as clock, logger, timer, and metrics,
prefer a variant-backed global configured once in `main` plus free functions
that dispatch through `lib::match`. Do not thread unused service references
through every constructor.

Good:

```cpp
using logger = std::variant<console_logger, file_logger, null_logger>;
inline logger global_logger{null_logger{}};

void log(level lvl, std::string_view msg)
{
  lib::match(global_logger, [&](auto& impl) { impl.log(lvl, msg); });
}
```

Reserve this pattern for services that truly cut across layers. Ordinary domain
dependencies still belong in signatures or adapters.

## Performance Discipline

Start with clear code and a known requirement. Optimize measured hot paths, not
hunches. Keep allocation, locality, and algorithmic complexity in view, but do
not contort cold code.

Hot-path swaps:

- bounded text: `lib::fixed_string<N>` instead of `std::string`;
- bounded vectors: `boost::container::static_vector<T, N>`;
- growing vectors: reserve once at startup;
- stored callbacks: `lib::inplace_function` instead of `std::function`;
- lookup containers: choose flat maps for locality, node maps when reference
  stability is load-bearing.

Document a non-obvious performance trade-off with the requirement or benchmark
that justifies it. Otherwise write the obvious version.

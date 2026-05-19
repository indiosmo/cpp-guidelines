# C++ Conventions Agent Context

Use this with `cpp-design-principles-agent-context.md`. This file is example
heavy on purpose: agents should copy these shapes and adapt the namespace
mapping to the target project.

Examples use:

- `lib::result<T>` for the project `boost::leaf::result<T>` alias;
- `lib::strong_type<T, Tag>` for the local strong-type helper;
- `lib::fixed_string<N>` for bounded hot-path text;
- `lib::inplace_function<Sig, Capacity>` for fixed-capacity callbacks;
- `lib::scope_exit` for rollback guards;
- `lib::match` for exhaustive variant visitation.

## Layout

Use the codebase's component layout. The common shape is:

```text
src/<component>/<component>/     public headers
src/<component>/src/             implementation
test/<component>/                tests
```

Functional code lives in the component root. Runtime wrappers live under
`<component>/runtime/`. Adapters that translate between domains carry both
domains in the name, for example `aor_fwll` or `aorfix_onixs_fix`.

## Formatting And Blank Lines

Use blank lines to mark phases in a function. Separate setup from a loop,
separate a multi-line loop or branch from the next statement, and separate the
final `return` from the block that computes its value.

Good:

```cpp
types::quantity total{0};

for (const auto& order : orders) {
  total += order.remaining_quantity;
}

return total;
```

Good:

```cpp
template <typename Alt>
auto request_id_of(const Alt& alt) -> std::optional<types::request_id>
{
  if constexpr (requires { alt.request_id; }) {
    return alt.request_id;
  }

  return std::nullopt;
}
```

Keep tightly coupled guard code together. A lookup and the `if` that checks it
are one unit, and a guard body can keep its diagnostic and early exit together.

Good:

```cpp
auto order_it = orders_.find(request.order_id);
if (order_it == orders_.end()) {
  log_warn("unknown order");
  return;
}
```

## Tests

Tests verify intended behavior, not implementation mechanics. Start from the
domain, protocol, contract, or bug report. Use the implementation to understand
the code path, but do not compute expected values by re-running the same logic
the test is supposed to check.

Good:

```cpp
TEST_CASE("rectangle - area", "[geometry]")
{
  const auto r = rectangle{.width = 6, .height = 4};
  CHECK(r.area() == 24);
}
```

Bad:

```cpp
TEST_CASE("rectangle - area", "[geometry]")
{
  const auto r = rectangle{.width = 6, .height = 4};
  CHECK(r.area() == r.width * r.height); // tautological
}
```

For bug fixes, add or tighten a test that fails before the fix. Confirm the
red state, then change production code until the test passes. For new
behavior, prefer writing the behavior test first so the contract shapes the
implementation.

When an expectation is non-obvious, name its source in the test comment or row
label: a protocol rule, public spec, fixture, black-box scenario, or bug
report. If the contract is type-level, test it at compile time with
`static_assert` / `STATIC_REQUIRE` instead of adding runtime scaffolding.

Keep test bodies dense with useful signal. Name the scenario, set up only the
data that matters, and assert outcomes a real defect would violate. Avoid
large suites of low-value cases that restate constructors, getters, or the
current algorithm without establishing intent.

Component tests should drive the functional surface directly. Construct domain
values, call the public function or stage entry point, and capture observable
outputs through callbacks or returned values. Do not spin runtime threads,
sockets, event loops, or SDK fixtures unless the test is explicitly about that
integration boundary.

Use factories, presets, and probes to keep setup out of the behavioral story.
Factories should let a test override only the fields that matter. Probes are
for narrow internal hygiene or branch setup that the public API should not
expose. For tests, sandboxes, and spikes, wire explicit noop callbacks for
irrelevant outputs; production wiring must assign real callbacks.

## Debugging

Before fixing a failing test or bug, identify the root cause. Capture the
exact failure, read the complete message or stack trace, reproduce it with the
smallest command or scenario, and trace the bad value or state backward to the
first component that produced it.

Good debugging loop:

```text
observe the symptom -> trace the source -> state one hypothesis
-> run one focused experiment -> write the failing regression test
-> fix the root cause -> run the relevant test and suite
```

Bad debugging loop:

```text
change code -> rerun test -> change more code -> rerun test
```

When the failure is deep in a call chain, add temporary logging before the
suspect call so the log captures the inputs that caused the failure. For
threaded or callback-heavy paths, capture enough context to identify the
request, branch, or test case that reached the bad state.

Use assertions for programmer invariants that should be impossible in correct
code. Return structured errors for invalid input or caller-visible failure
conditions. If a bug fix adds a runtime check, decide whether the stronger fix
is a refined type, parse-at-boundary conversion, boundary validation, rollback
guard, or explicit error path.

For flaky tests, suspect timing and leaked state first. Replace sleeps with
predicate waits, and wrap global providers, singleton alternatives, clocks,
environment variables, and other process-wide state in RAII guards so each
test restores the previous state even after a `REQUIRE` failure.

## Domain Types

Put a `types.hpp` in every domain from the start. Define domain types inside a
nested `types` namespace. Keep the `types::` qualifier even inside the domain;
it makes ownership visible and avoids field/type name collisions.

Good:

```cpp
namespace order_routing::types {

using user_id = lib::strong_type<std::uint64_t, struct UserIdTag>;
using user_order_id = lib::strong_type<std::uint64_t, struct UserOrderIdTag>;
using symbol = lib::strong_type<lib::fixed_string<8>, struct SymbolTag>;
using quantity = lib::strong_type<std::uint64_t, struct QuantityTag>;

enum class side : std::uint8_t { buy, sell };

} // namespace order_routing::types

namespace order_routing {

struct new_order {
  types::user_id user;
  types::user_order_id order_id;
  types::symbol instrument;
  types::side order_side;
  types::quantity order_quantity;
};

} // namespace order_routing
```

Bad:

```cpp
namespace order_routing {
using user_id = std::uint64_t;
using user_order_id = std::uint64_t;
using types::quantity; // Do not pull nested domain types upward.
}
```

Use strong types ergonomically. Do not unwrap and rewrap when the strong type
already supports the operation.

Good:

```cpp
types::quantity total{0};

for (const auto& order : orders) {
  total += order.remaining_quantity;
}

return total;
```

Bad:

```cpp
std::uint64_t total = 0;

for (const auto& order : orders) {
  total += order.remaining_quantity.get();
}

return types::quantity{total};
```

Only use `.get()` at a real boundary that requires the primitive.

## Strong-Type Ergonomics

Strong types are meant to reduce mismatch bugs without burying domain logic
under conversion ceremony. Use the strong type directly when the operation can
stay in the strong type.

Good:

```cpp
remaining_quantity -= fill_quantity;
auto residual = order_quantity - filled_quantity; // still a quantity
fmt::format("{}", order_id);
```

Bad:

```cpp
remaining_quantity = types::quantity{remaining_quantity.get() - fill_quantity.get()};
fmt::format("{}", order_id.get());
```

Same-type arithmetic preserves the strong type. Mixed expressions may fall back
to the underlying type when the local helper allows implicit underlying
references. Re-wrapping a mixed result must be explicit at the call site, which
makes domain mismatches reviewable.

Good:

```cpp
auto net = filled_quantity - canceled_quantity; // quantity
auto raw = filled_quantity - limit_price;       // underlying type, visibly mixed
```

Review carefully:

```cpp
auto suspicious = types::quantity{filled_quantity - limit_price};
```

That construction is allowed when the underlying type can construct the target,
but the names should make the mismatch stand out.

## Cross-Domain Strong-Type Construction

Do not write conversion helpers solely to move between two strong types with
the same safe representation. Construct the target strong type directly. If the
local strong-type helper permits the conversion, it compiles; if not, the
compiler rejects it.

Good:

```cpp
void submit_to_risk(routing::types::order_id routing_id)
{
  auto risk_id = risk::types::order_id{routing_id};
  risk_engine.submit(risk_id);
}
```

Bad:

```cpp
auto to_risk_order_id(routing::types::order_id id) -> risk::types::order_id
{
  return risk::types::order_id{id};
}

void submit_to_risk(routing::types::order_id routing_id)
{
  risk_engine.submit(to_risk_order_id(routing_id));
}
```

The helper adds machinery but no domain knowledge.

Use a named conversion helper when the mapping is semantic, lossy, fallible, or
shape-changing.

Good helpers:

```cpp
auto side_to_fix_char(types::side side) -> char;
auto parse_side(char value) -> lib::result<types::side>;
auto truncate_client_id(external::types::client_id id)
    -> lib::result<internal::types::client_id>;
```

Bad helper:

```cpp
auto to_risk_order_id(routing::types::order_id id) -> risk::types::order_id;
// Same representation, no validation, no semantic mapping.
```

## Designated Initializers

Use designated initializers for aggregates. Use trailing commas on multi-line
initializers. Omit fields that already have defaults.

Use brace elision for non-scalar fields such as strong types and containers;
use `=` for scalars.

Good:

```cpp
auto config = server_config{
  .listen_address = "0.0.0.0",
  .listen_port = 8080,
  .reuse_address = true,
  .backlog = 1024,
};

auto event = order_placed{
  .order_id{"O-12345"},
  .request_id{"R-98765"},
  .leaves_qty{100},
};
```

Bad:

```cpp
auto config = server_config{"0.0.0.0", 8080, true, 1024};

auto event = order_placed{
  .order_id = types::order_id{"O-12345"},
  .request_id = types::request_id{"R-98765"},
};
```

The field name already supplies the role; repeating the strong type usually
adds noise.

## Namespace Aliases

Never add namespace aliases or `using` declarations to headers. They leak into
includers and make lookup depend on include order.

In `.cpp` and test files, use aliases only when a file repeatedly mixes domain
vocabularies. Use short, stable aliases already common in the project.

Good:

```cpp
namespace md = market_data;
namespace me = matching_engine;
namespace rt = order_routing;

auto trade = md::trade{
  .user = md::types::user_id{request.user},
  .trade_price = md::types::price{request.limit_price},
};
```

Bad:

```cpp
namespace mkt = market_data;   // alternate alias for an existing convention
using namespace order_routing; // never in project code
using order_routing::types::price;
```

Within a function, local aliases for dense template or FSM namespaces are fine
when they improve readability:

```cpp
namespace sml = boost::sml;
namespace fsm = gateway::detail::session_state_machine_fsm;
```

## Const Placement

Use west const.

Good:

```cpp
void handle(const request& request);
for (const auto& order : orders) { /* ... */ }
```

Bad:

```cpp
void handle(request const& request);
for (auto const& order : orders) { /* ... */ }
```

## Comments

Use `/* ... */` for class, struct, header, and larger API documentation. Use
`//` for guiding comments inside function bodies. Comment present intent,
non-obvious behavior, and non-obvious ordering. Do not narrate obvious code,
describe old behavior, keep a trail of past decisions, record a migration
trail, or list what another layer handles.

Good class/header comment:

```cpp
/*
 * Typed requests produced by the decoder from wire bytes.
 * Market orders are signaled by limit_price == 0 and follow IOC semantics.
 */
struct new_order {
  types::user_id user;
  types::price limit_price;
};
```

Good block-level flow comments:

```cpp
// Reject duplicates and resolve the destination book up front.
BOOST_LEAF_CHECK(check_duplicate(incoming_key));
BOOST_LEAF_ASSIGN(auto* book_ptr, find_book(request.instrument));
auto& book = *book_ptr;

// Ack precedes trades so receivers see the order id before fills on it.
on_event(order_ack{
  .user{request.user},
  .order_id{request.order_id},
});

// Rest the residual: pool-allocate a node, link it into the book,
// register its identity for future cancels and duplicate checks.
order_node* node = allocate_node(order);
book.place(node);
resting_index_.emplace(incoming_key, node);
```

Bad comments:

```cpp
// Dispatch to the typed handle overload.
lib::match(cmd, [this](const auto& cmd) { handle(cmd); });

// Old implementation used a vector here.

// Dedup moved to the upstream stage.

// Increment the counter.
++counter;
```

Lean toward comments that summarize a block, even when the syntax is not
complex. A single sentence is faster to read than a loop, branch, or lambda
body. Consecutive block comments should read like the function's table of
contents.

Strongly comment non-obvious single lines: dense punctuation, designated
initializers across domains, chained calls, iterator invalidation, lifetime
contracts, performance-sensitive ordering, and protocol rules. Skip comments
for short, self-describing one-liners and repeatable local idioms. In
particular, a plain `lib::match` / `mil::match` dispatcher is usually
self-evident even if the syntax looks busy; comment the non-obvious choice
inside the visitor, not the fact that variant dispatch is happening.

If a precondition matters, phrase it as the invariant the code relies on:

```cpp
// precondition: session mutex is held; next_seq is read and incremented without locking here.
auto next = session_.next_seq++;
```

## Error Types

Each domain owns:

```text
<domain>/error_code.hpp   enum class error_code + std::error_code plumbing
<domain>/errors.hpp       structured error payloads with code()/what()
```

Good:

```cpp
namespace routing::errors {

struct unknown_order {
  types::order_id order_id;

  auto code() const -> std::error_code
  {
    return make_error_code(routing::error_code::unknown_order);
  }

  auto what() const -> std::string
  {
    return fmt::format("unknown order {}", order_id);
  }
};

} // namespace routing::errors
```

Bad:

```cpp
return lib::make_leaf_error(lib::error_code::generic_error,
                            "unknown routing order"); // loses typed context
```

Generic errors are fine for generic infrastructure failures such as bad config.
Domain failures should carry structured domain context.

## Result Flow

Inside a domain, compose fallible steps with `BOOST_LEAF_ASSIGN` and
`BOOST_LEAF_CHECK`.

Good:

```cpp
lib::result<void> route(const new_order& request)
{
  BOOST_LEAF_ASSIGN(const auto& sink, select_sink(request.route));
  BOOST_LEAF_CHECK(orders_.process(request));
  on_routed(routed_order{.sink{sink}, .request = request});
  return {};
}
```

Bad:

```cpp
bool route(const new_order& request)
{
  auto sink = select_sink(request.route);
  if (!sink) {
    log("no sink");
    return false;
  }
  if (!orders_.process(request)) {
    log("order failed");
    return false;
  }
  return true;
}
```

At public boundaries, consume `lib::result` with `boost::leaf::try_handle_all`
or `try_handle_some`, then return the boundary's vocabulary.

Good:

```cpp
void engine::handle(const request& request)
{
  boost::leaf::try_handle_all(
    [&]() -> lib::result<void> {
      BOOST_LEAF_CHECK(handle_impl(request));
      return {};
    },
    [&](lib::match_error<errors::duplicate_order>) {
      log_warn("duplicate order {}", request.order_id);
    },
    LIB_RESULT_CATCH_ALL(log_error("unhandled engine error")));
}
```

Bad:

```cpp
lib::result<void> engine::handle(const request& request); // leaks LEAF boundary
```

Allowed exception: a utility algorithm whose public contract is "this fallible
algorithm returns the project result type."

## Error Handler Ordering

LEAF handlers are ordered. Put specific, intentional cases first and catch-alls
last.

Good order:

```cpp
return std::tuple_cat(
  std::make_tuple(
    [&](match_errors<duplicate_order, duplicate_request> err) {
      log_precondition(err);
      return drop_request();
    }),
  std::forward<Handlers>(handlers)...,
  std::make_tuple(
    [&](match_errors<invalid_field, unknown_route> err) {
      return reject(request, err.matched);
    },
    LIB_RESULT_CATCH_ALL(return reject_internal(request);)));
```

Bad:

```cpp
return std::make_tuple(
  LIB_RESULT_CATCH_ALL(return reject_internal(request);),
  [&](match_error<invalid_field> err) { return reject(request, err.value()); });
// The catch-all shadows the specific handler.
```

## Exhaustiveness

For `enum class`, switch without `default` so the compiler catches missing
enumerators. Return a structured error after the switch for corrupted or
deserialized impossible values.

Good:

```cpp
auto to_wire(types::side side) -> lib::result<char>
{
  switch (side) {
    case types::side::buy: return '1';
    case types::side::sell: return '2';
  }

  return lib::make_leaf_error(errors::invalid_field_value{.field{"side"}});
}
```

For untyped boundary values, use `default` because the compiler cannot help:

```cpp
auto parse_side(char value) -> lib::result<types::side>
{
  switch (value) {
    case '1': return types::side::buy;
    case '2': return types::side::sell;
    default:
      return lib::make_leaf_error(errors::invalid_field_value{
        .field{"side"},
        .value{std::string{value}},
      });
  }
}
```

## Variants

Use `lib::match` for domain decisions. Add one arm per meaningful alternative.

Good:

```cpp
void send(const request& req)
{
  lib::match(req, [this](const auto& typed) { handle(typed); });
}
```

Bad:

```cpp
if (std::holds_alternative<new_order>(req)) {
  handle(std::get<new_order>(req));
} else if (std::holds_alternative<cancel_order>(req)) {
  handle(std::get<cancel_order>(req));
} else {
  std::abort();
}
```

Use `lib::match_partial` only for intentional subscriptions where ignored
alternatives are no-ops.

## Pipeline Callbacks

Stage callbacks are public wiring fields, usually `on_<event>`, stored as
`lib::inplace_function`. Production wiring must assign every non-defaulted
callback before it can fire.

Good:

```cpp
class session {
public:
  void send(datagram_view bytes);

  lib::inplace_function<void(request&&)> on_request;
  lib::inplace_function<void(rejection&&)> on_rejected;
};

session.on_request = [&engine, &engine_loop](request&& req) {
  engine_loop.post([&engine, req = std::move(req)]() mutable {
    engine.send(std::move(req));
  });
};
```

Bad:

```cpp
class session {
public:
  engine* engine_; // session now knows its consumer and thread policy
};
```

For tests, sandboxes, and constructor-failure cleanup only, wire explicit noop
callbacks for irrelevant outputs:

```cpp
stage.on_rejected = [](rejection&&) {};
```

Do not hide missing production wiring with default noops unless the callback is
truly optional.

## Scope Guards

Use scope guards when partial mutation must be rolled back on any failure path.
Dismiss guards only after the whole operation commits.

Good:

```cpp
orders_.emplace(request.order_id, build_order_state(request));
auto undo_order = lib::scope_exit([&] { orders_.erase(request.order_id); });

requests_.emplace(request.request_id, build_request_state(request));
auto undo_request = lib::scope_exit([&] { requests_.erase(request.request_id); });

BOOST_LEAF_CHECK(limits_.reserve(request));

undo_request.dismiss();
undo_order.dismiss();
return {};
```

Bad:

```cpp
orders_.emplace(request.order_id, build_order_state(request));
BOOST_LEAF_CHECK(limits_.reserve(request)); // failure leaves order inserted
requests_.emplace(request.request_id, build_request_state(request));
```

## Runtime Posts

Post across threads. Capture moved messages by value. Keep closures small
enough for the project's inplace-function task storage.

Good:

```cpp
output_loop.post([this, ev = std::move(ev)]() mutable {
  publisher_.send(std::move(ev));
});
```

Bad:

```cpp
output_loop.post([&] {
  publisher_.send(ev); // ev may dangle after the caller returns
});
```

Do not wait synchronously for a result from the target loop. Model replies as
events posted back.

## State Machines

Put Boost.SML state machine vocabulary in a detail namespace. Use `ev_*` event
tags, `st_*` state tags, an `actions` struct of injected callbacks, and a
transition table. The owner stores actions before the machine.

Good:

```cpp
namespace gateway::detail::session_state_machine_fsm {

struct ev_connect {};
struct ev_closed {};
struct st_closed {};
struct st_connecting {};

struct actions {
  lib::inplace_function<void()> async_connect;
};

struct transitions {
  auto operator()() const noexcept
  {
    namespace sml = boost::sml;
    return sml::make_transition_table(
      *sml::state<st_closed> + sml::event<ev_connect> = sml::state<st_connecting>,
      sml::state<st_connecting> + sml::on_entry<sml::_> /
        [](actions& a) { a.async_connect(); });
  }
};

} // namespace gateway::detail::session_state_machine_fsm
```

Bad:

```cpp
bool connected_ = false;
bool connecting_ = false;
bool retrying_ = false;
// Illegal combinations are now representable.
```

For non-trivial machines, put a PlantUML `/* @startuml ... @enduml */` block
immediately above the transition table and update it with every table change.

## Concepts And Requires

Use concepts for public template contracts. Use inline `requires` for local
capability dispatch.

Good:

```cpp
template <typename T>
concept ErrorData = requires(T t) {
  { t.code() } -> std::same_as<std::error_code>;
  { t.what() } -> std::same_as<std::string>;
};

template <typename Alt>
auto request_id_of(const Alt& alt) -> std::optional<types::request_id>
{
  if constexpr (requires { alt.request_id; }) {
    return alt.request_id;
  }

  return std::nullopt;
}
```

Bad:

```cpp
template <typename Alt>
auto request_id_of(const Alt& alt)
{
  if constexpr (std::is_same_v<Alt, placed> || std::is_same_v<Alt, canceled>) {
    return alt.request_id;
  }
}
```

Dispatch on capability when the data shape is what matters.

## Hot-Path Storage

Use bounded storage on measured or explicitly constrained hot paths.

Good:

```cpp
using symbol = lib::strong_type<lib::fixed_string<16>, struct SymbolTag>;

std::vector<order> orders_;
orders_.reserve(config.max_orders);

lib::inplace_function<void(message&&), 256> on_message;
```

Bad:

```cpp
using symbol = std::string;       // allocates per hot-path value
std::vector<order> orders_;       // grows during request handling
std::function<void(message&&)> cb; // may allocate when assigned
```

Use node maps when references or pointers must survive inserts and rehashes.
Use flat maps for cache-friendly lookup tables where no long-lived reference is
kept.

## Result Pipeline Extraction

When a function has three or more sequential fallible steps, extract named
helpers and let the body read as the success path.

Before:

```cpp
void handle(const cancel_order& request)
{
  auto order_it = orders_.find(request.order_id);
  if (order_it == orders_.end()) {
    log_warn("unknown order");
    return;
  }

  auto book_it = books_.find(order_it->second.symbol);
  if (book_it == books_.end()) {
    log_error("missing book");
    return;
  }

  cancel(book_it->second, order_it->second);
}
```

After:

```cpp
lib::result<void> handle_cancel_impl(const cancel_order& request)
{
  BOOST_LEAF_ASSIGN(auto& order, find_order(request.order_id));
  BOOST_LEAF_ASSIGN(auto& book, find_book(order.symbol));
  BOOST_LEAF_CHECK(cancel(book, order));
  return {};
}

void handle(const cancel_order& request)
{
  boost::leaf::try_handle_all(
    [&] { return handle_cancel_impl(request); },
    [&](lib::match_error<errors::unknown_order>) { log_warn("unknown order"); },
    [&](lib::match_error<errors::missing_book>) { log_error("missing book"); },
    LIB_RESULT_CATCH_ALL(log_error("unhandled cancel error")));
}
```

Skip this extraction for one or two obvious checks, or when the scaffolding
would create more error types than clarity.

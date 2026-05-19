# C++ Conventions -- Examples

Good/bad code pairs that illustrate the convention rules in
[`cpp-agent-context.md`](cpp-agent-context.md). Load this file on
demand when the rule alone is not enough to shape an edit. Section
titles match the rule sections in the agent context. Examples use
`lib::` placeholders -- substitute the local project vocabulary.

## Tests

Good:

```cpp
TEST_CASE("rectangle - area", "[geometry]")
{
  const auto r = rectangle{.width = 6, .height = 4};
  CHECK(r.area() == 24);
}
```

Bad -- expected value rederived from the implementation:

```cpp
TEST_CASE("rectangle - area", "[geometry]")
{
  const auto r = rectangle{.width = 6, .height = 4};
  CHECK(r.area() == r.width * r.height);
}
```

## Debugging

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

## Domain Types

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

Strong-type ergonomics inside the domain. Good:

```cpp
types::quantity total{0};

for (const auto& order : orders) {
  total += order.remaining_quantity;
}

return total;
```

Bad -- unwrap and rewrap:

```cpp
std::uint64_t total = 0;

for (const auto& order : orders) {
  total += order.remaining_quantity.get();
}

return types::quantity{total};
```

## Strong-Type Ergonomics

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

Mixed expressions, explicit rewrapping:

```cpp
auto net = filled_quantity - canceled_quantity; // quantity
auto raw = filled_quantity - limit_price;       // underlying type, visibly mixed
auto suspicious = types::quantity{filled_quantity - limit_price}; // review carefully
```

## Cross-Domain Strong-Type Construction

Good -- direct construction:

```cpp
void submit_to_risk(routing::types::order_id routing_id)
{
  auto risk_id = risk::types::order_id{routing_id};
  risk_engine.submit(risk_id);
}
```

Bad -- helper with no domain knowledge:

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

Helpers that earn their name:

```cpp
auto side_to_fix_char(types::side side) -> char;
auto parse_side(char value) -> lib::result<types::side>;
auto truncate_client_id(external::types::client_id id)
    -> lib::result<internal::types::client_id>;
```

## Designated Initializers

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

Bad -- positional, or strong-type rewrap noise:

```cpp
auto config = server_config{"0.0.0.0", 8080, true, 1024};

auto event = order_placed{
  .order_id = types::order_id{"O-12345"},
  .request_id = types::request_id{"R-98765"},
};
```

## Namespace Aliases

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

Local aliases inside a function are fine:

```cpp
namespace sml = boost::sml;
namespace fsm = gateway::detail::session_state_machine_fsm;
```

## Const Placement

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

Precondition phrased as the invariant the code relies on:

```cpp
// precondition: session mutex is held; next_seq is read and incremented without locking here.
auto next = session_.next_seq++;
```

## Error Types

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

Bad -- loses typed context:

```cpp
return lib::make_leaf_error(lib::error_code::generic_error,
                            "unknown routing order");
```

## Result Flow

Good -- inside the domain:

```cpp
lib::result<void> route(const new_order& request)
{
  BOOST_LEAF_ASSIGN(const auto& sink, select_sink(request.route));
  BOOST_LEAF_CHECK(orders_.process(request));
  on_routed(routed_order{.sink{sink}, .request = request});
  return {};
}
```

Bad -- bool plus side-channel logging:

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

Good -- at a public boundary:

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

Bad -- LEAF boundary leaked through the signature:

```cpp
lib::result<void> engine::handle(const request& request);
```

## Error Handler Ordering

Good -- specific before catch-all:

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

Bad -- catch-all shadows the specific handler:

```cpp
return std::make_tuple(
  LIB_RESULT_CATCH_ALL(return reject_internal(request);),
  [&](match_error<invalid_field> err) { return reject(request, err.value()); });
```

## Exhaustiveness

Good -- typed enum, no `default`:

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

Good -- untyped boundary value, `default` required:

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

## Pipeline Callbacks

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

Bad -- session knows its consumer and thread policy:

```cpp
class session {
public:
  engine* engine_;
};
```

Test/sandbox wiring:

```cpp
stage.on_rejected = [](rejection&&) {};
```

## Scope Guards

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

Bad -- failure leaves order inserted:

```cpp
orders_.emplace(request.order_id, build_order_state(request));
BOOST_LEAF_CHECK(limits_.reserve(request));
requests_.emplace(request.request_id, build_request_state(request));
```

## Runtime Posts

Good:

```cpp
output_loop.post([this, ev = std::move(ev)]() mutable {
  publisher_.send(std::move(ev));
});
```

Bad -- ev may dangle after the caller returns:

```cpp
output_loop.post([&] {
  publisher_.send(ev);
});
```

## State Machines

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

Bad -- correlated booleans:

```cpp
bool connected_ = false;
bool connecting_ = false;
bool retrying_ = false;
```

## Concepts And Requires

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

Bad -- dispatch on type name:

```cpp
template <typename Alt>
auto request_id_of(const Alt& alt)
{
  if constexpr (std::is_same_v<Alt, placed> || std::is_same_v<Alt, canceled>) {
    return alt.request_id;
  }
}
```

## Hot-Path Storage

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

## Result Pipeline Extraction

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

# C++ Design Principles -- Examples

Good/bad code pairs that illustrate the rules in
[`cpp-design-principles-agent-context.md`](cpp-design-principles-agent-context.md).
Load this file on demand when the prose alone is not enough to guide a
concrete shape. Examples use `lib::` placeholders -- substitute the local
project vocabulary.

## Domain Ownership

Same safe representation, no helper:

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

Real conversion, helper earns its name:

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

Bad -- helper adds no domain knowledge:

```cpp
auto to_risk_order_id(routing::types::order_id id) -> risk::types::order_id
{
  return risk::types::order_id{id};
}
```

## Forward Dependencies

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

## Types Carry Proof

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

## Functional Core, Imperative Shell

Good -- pure value function:

```cpp
auto apply_fill(order_state order, fill fill) -> order_state;
```

Bad -- effects buried in domain code:

```cpp
void apply_fill(order_state& order, fill fill)
{
  global_logger.info("fill {}", fill.id);
  background_executor.post([&] { persist(order); });
}
```

Good -- effects at the runtime boundary:

```cpp
class risk_check {
public:
  auto check(const order& order, const limits& limits) -> lib::result<decision>;
};

// Near main: choose the provider, fetch the limits, call the pure component.
auto limits = limits_provider.load(account);
BOOST_LEAF_ASSIGN(auto decision, risk.check(order, limits));
```

Bad -- I/O hidden inside the domain component:

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

## Pipelines

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

## Runtime And Threads

Good:

```cpp
consumer_loop.post([&consumer, event = std::move(event)]() mutable {
  consumer.send(std::move(event));
});
```

Bad -- direct cross-thread call into unsynchronized state:

```cpp
consumer.send(std::move(event));
```

## Error Handling

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

Bad -- untyped, easy to ignore, hard to route:

```cpp
bool process(std::string* error_text);
```

## Invariants And Rollback

Commit-at-end:

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

Scope guard when a later fallible step must observe the mutation:

```cpp
orders_.emplace(order.id, order_state);
auto undo_order = lib::scope_exit([&] { orders_.erase(order.id); });

BOOST_LEAF_CHECK(limits_.reserve(order));

undo_order.dismiss();
```

## Declarative Style

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

## Guiding Comments

Good -- block-level intent:

```cpp
// reject duplicates and resolve the destination book up front.
BOOST_LEAF_CHECK(check_duplicate(incoming_key));
BOOST_LEAF_ASSIGN(auto* book_ptr, find_book(request.instrument));
auto& book = *book_ptr;

// ack precedes trades so receivers see the order id before fills on it.
on_event(order_ack{.user = user, .order_id = order_id});
```

Bad -- narrating a self-evident dispatch:

```cpp
// Dispatch to the typed handle overload.
lib::match(cmd, [this](const auto& typed) { handle(typed); });
```

## Variants, Concepts, And Templates

Capability dispatch over `std::variant`:

```cpp
return lib::match(event, [](const auto& alt) -> std::optional<types::request_id> {
  if constexpr (requires { alt.request_id; }) {
    return alt.request_id;
  }

  return std::nullopt;
});
```

## State Machines

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

## Cross-Cutting Services

Good:

```cpp
using logger = std::variant<console_logger, file_logger, null_logger>;
inline logger global_logger{null_logger{}};

void log(level lvl, std::string_view msg)
{
  lib::match(global_logger, [&](auto& impl) { impl.log(lvl, msg); });
}
```

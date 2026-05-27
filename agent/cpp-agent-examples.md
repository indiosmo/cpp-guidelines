# C++ Agent Examples

Good/bad code pairs that illustrate the rules in
[`cpp-agent-context.md`](cpp-agent-context.md). Section titles match
the rule sections in the context file. Load this file on demand when
the rule alone is not enough to shape an edit. Examples use `lib::`
placeholders -- substitute the local project vocabulary.

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

## Domain Ownership

Domain `types.hpp` with strong types and an enum class. Good:

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

Bad -- raw primitives in the domain, or domain types pulled upward:

```cpp
namespace order_routing {
using user_id = std::uint64_t;
using user_order_id = std::uint64_t;
using types::quantity; // Do not pull nested domain types upward.
}
```

Two domains owning the same safe representation -- no helper, just
direct cross-domain construction:

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
  risk_engine.submit(risk_id);
}
```

Bad -- helper adds no domain knowledge:

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

Helpers that earn their name -- semantic, lossy, fallible, or
shape-changing mappings:

```cpp
auto side_to_fix_char(types::side side) -> char;
auto parse_side(char value) -> lib::result<types::side>;
auto truncate_client_id(external::types::client_id id)
    -> lib::result<internal::types::client_id>;
```

## Forward Dependencies

Good -- each function takes what it needs and returns what it
produces:

```cpp
auto untie(const table& data) -> column;
auto rank(const table& data, const column& tiebreaker) -> table;

auto score(const table& data) -> table
{
  return rank(data, untie(data));
}
```

Bad -- implicit residue, ordering known only to the author:

```cpp
struct working_set {
  table data;
  std::optional<column> tiebreaker; // set by untie(), consumed by rank()
};

void untie(working_set&);
void rank(working_set&); // only valid after untie()
```

## Types Carry Proof

Good -- parse once, then trust the refined types:

```cpp
auto parse_symbol(std::string_view raw) -> lib::result<types::symbol>;

void place_order(types::account account, types::symbol symbol, types::quantity qty);
```

Bad -- every downstream caller must remember which strings were
validated:

```cpp
auto validate_symbol(std::string_view raw) -> bool;

void place_order(std::string_view account, std::string_view symbol, int qty);
```

## Strong-Type Ergonomics

Good -- same-type arithmetic stays in the strong type; formatting
goes through the strong type:

```cpp
remaining_quantity -= fill_quantity;
auto residual = order_quantity - filled_quantity; // still a quantity
fmt::format("{}", order_id);
```

Bad -- unwrap and rewrap noise:

```cpp
remaining_quantity = types::quantity{remaining_quantity.get() - fill_quantity.get()};
fmt::format("{}", order_id.get());
```

Same shape inside a loop. Good:

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

Mixed-domain expressions stay visible, explicit rewrapping marks the
review point:

```cpp
auto net = filled_quantity - canceled_quantity; // quantity
auto raw = filled_quantity - limit_price;       // underlying type, visibly mixed
auto suspicious = types::quantity{filled_quantity - limit_price}; // review carefully
```

Member access goes through the wrapper's arrow. Good -- chain through
the strong type to a member of the underlying:

```cpp
// secinfo.contract_multiplier : lib::strong_type<lib::decimal, ContractMultiplierTag>
const auto multiplier = secinfo.contract_multiplier->as_double();

// order_state.source_name : lib::strong_type<lib::fixed_string<42>, SourceNameTag>
log_info("source {}", order_state.source_name->to_string_view());
```

Bad -- `.get().member()` adds an unwrap where the arrow already names
the underlying member operation:

```cpp
const auto multiplier = secinfo.contract_multiplier.get().as_double();
log_info("source {}", order_state.source_name.get().to_string_view());
```

Bad -- assignment through `.get()` tries to mutate through a read-only
view:

```cpp
order.symbol.get() = "PETR4";
```

Good -- construct a new strong-typed value and assign that whole:

```cpp
order.symbol = types::symbol{"PETR4"};
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

Good -- stage exposes inbound member functions and outbound callback
fields; wiring near `main` decides the topology:

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

Test or sandbox wiring -- explicit noop for outputs irrelevant to the
scenario:

```cpp
stage.on_rejected = [](rejection&&) {};
```

## Runtime And Threads

Good -- capture posted work by value:

```cpp
output_loop.post([this, ev = std::move(ev)]() mutable {
  publisher_.send(std::move(ev));
});
```

Bad -- `ev` may dangle after the caller returns:

```cpp
output_loop.post([&] {
  publisher_.send(ev);
});
```

Bad -- direct cross-thread call into unsynchronized state:

```cpp
consumer.send(std::move(event));
```

## Error Handling

Structured error payload with `code()` and `what()`. Good:

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

Result flow inside the domain. Good:

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

Public boundary consumes the result and returns caller vocabulary.
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

Bad -- LEAF boundary leaked through the signature:

```cpp
lib::result<void> engine::handle(const request& request);
```

Handler ordering: specific before catch-all. Good:

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

Exhaustiveness for typed enums -- no `default`:

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

Untyped boundary values -- `default` required:

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

Extracting a result pipeline -- before:

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

Result unwrap goes through the propagation macros. Good -- macro
unwraps the result and binds the success value:

```cpp
BOOST_LEAF_ASSIGN(const auto& token, format_flag_set(set, exec_inst_table));
process(token);
```

Good -- `BOOST_LEAF_CHECK` for the void case:

```cpp
BOOST_LEAF_CHECK(commit_order(request));
```

Bad -- `result->member` is unchecked. On an error result,
`lib::result::operator->` returns `nullptr`, and the chained call
dereferences null:

```cpp
fmt::format("{}", format_flag_set(set, exec_inst_table)->to_string_view());
```

Bad -- `.value()` throws on error at a site that does not expect to
catch:

```cpp
const auto wire = format_flag_set(set, exec_inst_table).value().to_string_view();
```

Combined `lib::result<lib::strong_type<T, Tag>>` -- unwrap the result
with the LEAF macro first, then chain through the strong type's arrow.
Good:

```cpp
BOOST_LEAF_ASSIGN(const auto& order_id, tracker.get_order_id(client_id));
log_info("order {}", order_id->to_string_view());
```

Bad -- `.value().get()` stacks both unidiomatic unwraps:

```cpp
const auto id = tracker.get_order_id(client_id).value().get().to_string_view();
```

## Invariants And Rollback

Commit-at-end -- mutate a copy, swap in on success:

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

Scope guard when a later fallible step must observe the mutation.
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

## Declarative Style

Good -- stage derived values, name predicates:

```cpp
const bool has_residual = order.remaining_quantity > 0;
const bool is_market = order.limit_price == 0;
const bool should_rest = has_residual && !is_market;

if (should_rest) {
  rest(order);
}
```

Bad -- conditions inline, harder to read:

```cpp
if (order.remaining_quantity > 0 && !(order.limit_price == 0)) {
  rest(order);
}
```

Good -- bind an optional value where the branch proves it is usable:

```cpp
for (const auto& fill : fills) {
  if (const auto resting_order = order_book.find_resting(fill.order_id);
      resting_order) {
    apply_fill(*resting_order, fill);
  }
}
```

Bad -- assign, guard, then use:

```cpp
for (const auto& fill : fills) {
  const auto resting_order = order_book.find_resting(fill.order_id);
  if (!resting_order) {
    continue;
  }

  apply_fill(*resting_order, fill);
}
```

## Variants, Concepts, And Templates

`lib::match` for variant dispatch. Good:

```cpp
void send(const request& req)
{
  lib::match(req, [this](const auto& typed) { handle(typed); });
}
```

Bad -- manual `holds_alternative` chain:

```cpp
if (std::holds_alternative<new_order>(req)) {
  handle(std::get<new_order>(req));
} else if (std::holds_alternative<cancel_order>(req)) {
  handle(std::get<cancel_order>(req));
} else {
  std::abort();
}
```

Concept for a public template contract:

```cpp
template <typename T>
concept ErrorData = requires(T t) {
  { t.code() } -> std::same_as<std::error_code>;
  { t.what() } -> std::same_as<std::string>;
};
```

Capability dispatch with `if constexpr (requires { ... })`. Good:

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

Constrain a template that assumes a shape. The body of
`make_error_handlers` reads `Outcome::result_type`, calls
`Outcome::error(...)` and `Outcome::fault()`; that contract belongs in
the signature.

Bad -- bare `typename`, contract hidden in the body:

```cpp
template <typename Outcome, lib::Tuple... Handlers>
auto make_error_handlers(const new_order& req, Handlers&&... handlers);
```

Good -- concept names the contract; misuse fails at the call site:

```cpp
template <lib::Outcome Outcome, lib::Tuple... Handlers>
auto make_error_handlers(const new_order& req, Handlers&&... handlers);
```

The deduction-friendly call site is unchanged either way:

```cpp
// Caller writes the outcome alias once; result/error types are deduced.
auto handlers = make_error_handlers<order_outcome>(req);
```

Concept paired with a small trait when the parameter must be a specific
class-template specialisation:

```cpp
namespace detail {

template <typename T>
struct is_outcome : std::false_type {};

template <typename Result, typename Error>
struct is_outcome<outcome<Result, Error>> : std::true_type {};

template <typename T>
inline constexpr bool is_outcome_v = is_outcome<T>::value;

} // namespace detail

template <typename T>
concept Outcome =
  detail::is_outcome_v<std::remove_cvref_t<T>>
  && requires {
       typename std::remove_cvref_t<T>::result_type;
       typename std::remove_cvref_t<T>::error_type;
       { std::remove_cvref_t<T>::error(
           std::declval<typename std::remove_cvref_t<T>::error_type>()) }
         -> std::same_as<std::remove_cvref_t<T>>;
       { std::remove_cvref_t<T>::fault() }
         -> std::same_as<std::remove_cvref_t<T>>;
     };
```

## State Machines

Good -- Boost.SML in a detail namespace, with event/state tags and an
`actions` struct:

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

## Cross-Cutting Services

Good -- variant-backed global configured once in `main`, free
functions dispatch through `lib::match`:

```cpp
using logger = std::variant<console_logger, file_logger, null_logger>;
inline logger global_logger{null_logger{}};

void log(level lvl, std::string_view msg)
{
  lib::match(global_logger, [&](auto& impl) { impl.log(lvl, msg); });
}
```

## Performance Discipline

Good -- bounded storage on hot paths, reserved once at startup:

```cpp
using symbol = lib::strong_type<lib::fixed_string<16>, struct SymbolTag>;

std::vector<order> orders_;
orders_.reserve(config.max_orders);

lib::inplace_function<void(message&&), 256> on_message;
```

Bad -- allocates per hot-path value, grows during request handling,
may allocate when assigned:

```cpp
using symbol = std::string;
std::vector<order> orders_;
std::function<void(message&&)> cb;
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

Good block-level flow comments -- the function's table of contents:

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

## Namespace Aliases

Good -- short, stable aliases at the top of a `.cpp` or test file:

```cpp
namespace md = market_data;
namespace me = matching_engine;
namespace rt = order_routing;

auto trade = md::trade{
  .user = md::types::user_id{request.user},
  .trade_price = md::types::price{request.limit_price},
};
```

Bad -- alternate alias for an existing convention, blanket `using
namespace`, or `using` declaration pulling a domain type into the
local scope:

```cpp
namespace mkt = market_data;
using namespace order_routing;
using order_routing::types::price;
```

Local aliases inside a function for dense template or FSM namespaces:

```cpp
namespace sml = boost::sml;
namespace fsm = gateway::detail::session_state_machine_fsm;
```

## Const Placement

Good -- west const:

```cpp
void handle(const request& request);
for (const auto& order : orders) { /* ... */ }
```

Bad -- east const:

```cpp
void handle(request const& request);
for (auto const& order : orders) { /* ... */ }
```

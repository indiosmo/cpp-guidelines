# Compile-time correctness

A mistake that becomes a compile error costs minutes; the same mistake in
production costs days. The patterns in this file move checks the reader
would otherwise have to do by hand into the type system. Strong types stop
parameter swaps at the call site. Parsing turns untyped input into refined
types at the boundary, so the rest of the program operates on values that
cannot misrepresent. Plain aggregates keep move semantics and reflection
working. Exhaustive switches enlist the compiler to flag missing cases.
Designated initializers make struct construction self-documenting and
refactor-safe.

## Strong typing

Wrap primitives in strong types when adjacent parameters share a primitive type. The compiler then catches argument swaps.

```cpp
// BAD - easy to confuse parameters
void place_order(std::string_view account, std::string_view symbol, int qty, double price);

place_order("AAPL", "ACCT1", 100, 150.0);  // oops: account and symbol swapped, compiles fine
```

```cpp
// GOOD - distinct types per role; the compiler catches the swap
using account_id = lib::strong_type<std::string,  struct AccountIdTag>;
using symbol     = lib::strong_type<std::string,  struct SymbolTag>;
using order_qty  = lib::strong_type<std::int32_t, struct OrderQtyTag>;
using price      = lib::strong_type<double,       struct PriceTag>;

void place_order(account_id account, symbol sym, order_qty qty, price px);

place_order(symbol{"AAPL"}, account_id{"ACCT1"}, order_qty{100}, price{150.0});
//          ^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^
// compile error: types do not match parameter order
```

`lib::strong_type<T, Tag>` gives each alias its own identity even when two aliases share the same underlying type. Libraries like `boost::strong_typedef` or `NamedType` provide this shape, or write a small in-house helper.

## Per-domain `types` namespace

Every domain defines its types in a `types.hpp` header inside a nested
`types` namespace from the start, regardless of how few there are. The
`types::` qualifier sidesteps name conflicts between a type and a field of
the same name, so structs read naturally without a renaming dance.

```cpp
#include "domain/types.hpp"

namespace domain {

struct message {
  types::symbol symbol;  // works fine; no naming gymnastics required
};

}  // namespace domain
```

Without the nested namespace, the field `symbol` would shadow the type
`symbol` and the struct would need an ugly workaround (`symbol_ symbol;`,
`Symbol symbol;`, or a fully-qualified `::domain::symbol`).

## Parse, don't validate

The cheapest check is the one that never runs because the invalid value
cannot be constructed. A `types::symbol` whose only public constructor takes
a `std::string_view` and returns `std::expected<types::symbol, error_code>`
moves the shape check to a single site -- the factory -- and every
downstream user of the type inherits the proof for free.

This is the difference between *parsing* and *validating*. A validator
returns `bool` (or throws): it consumes a value, produces a fact, and
discards the fact the moment control returns. A parser returns
`std::expected<refined_type, error_code>`: it consumes a value, produces *a
value that encodes the fact*, and the fact rides along with every
subsequent use.

```cpp
// BAD - validate; the fact is gone the moment validate_symbol returns
auto validate_symbol(std::string_view s) -> bool;

void place_order(std::string_view symbol, ...)
{
  if (!validate_symbol(symbol)) { /* handle bad symbol */ }
  // every downstream function that takes `symbol` as std::string_view
  // either re-validates or takes the prior check on faith
}
```

```cpp
// GOOD - parse; the type carries the proof
auto parse_symbol(std::string_view s)
    -> std::expected<types::symbol, error_code>;

void place_order(types::symbol sym, ...)
{
  // sym is non-empty and well-formed by construction; nothing
  // downstream needs to re-check
}
```

Parsing happens at the program boundary. Once the value is past it, the
function signature itself is the check: a caller that hands `place_order`
something other than a `types::symbol` does not compile, and no runtime
code runs to catch it. The error lands at the point the caller can do
something about it -- not five frames deep, where the original input is
out of scope.

The type system does not reach state-dependent invariants (whether the
market is open, whether an account holds funds) or values that re-enter
from untyped sources (bytes read back from a JSON file or shared memory
must be re-parsed). Those checks live at runtime; see
`../cpp-debugging-principles/defense-in-depth.md` for the layered runtime
strategy that complements parsing at the boundary.

## Data structs as plain aggregates

Structs that exist to carry data should be aggregates: public fields, no
custom constructors. Do not mark data-struct fields `const`. A `const`
non-static member breaks too many things in silently-painful ways:

- **Reflection and serialization libraries.** `boost::pfr`,
  `nlohmann::json`'s automatic reflection, and similar tools rely on the type
  being a plain aggregate; a `const` member disqualifies it.
- **Move semantics.** A `const` member cannot be assigned, so the implicitly
  generated move-assignment operator is deleted. Any container of these
  structs becomes painfully expensive (or just won't compile when sorted,
  resized, or moved into a slot).
- **Triviality.** `std::is_trivially_copyable_v` becomes false, disabling `memcpy`-based serialization.

```cpp
// BAD - const member breaks pfr/json reflection and move-assign
struct order_event {
  const types::exec_type exec_type;  // looks principled, isn't
  types::order_id        order_id;
  types::price           price;
};
```

```cpp
// GOOD - plain aggregate
struct order_event {
  types::exec_type exec_type;
  types::order_id  order_id;
  types::price     price;
};
```

When a value is invariant across all instances of a type, promote it to
`static constexpr`. Static members do not participate in object layout, do
not interact with move semantics, and are visible to reflection as a class
property rather than an instance field.

```cpp
// GOOD - the tag is part of the type, not the instance
struct order_placed {
  static constexpr auto exec_type = types::exec_type::placed;

  types::order_id   order_id;
  types::request_id request_id;
};

struct order_canceled {
  static constexpr auto exec_type = types::exec_type::canceled;

  types::order_id   order_id;
  types::request_id request_id;
};

using order_event = std::variant<order_placed, order_canceled>;
```

Serializers read `T::exec_type` as the discriminator; deserializers dispatch
on it to pick the variant alternative. For the broader principle of making
illegal states unrepresentable through type design, see `invariants.md`.

The `if constexpr (requires { ... })` pattern in `templates.md` shows how a
generic visitor extracts shared fields from these alternatives without
hard-coding the alternative types.

## Exhaustiveness checking

Prefer `enum class` over bare integers, and prefer a `switch` with no
`default` over an `if/else` chain. The switch enlists the compiler: a new
enumerator produces a `-Wswitch` warning at every site that does not
handle it.

```cpp
enum class side { buy, sell, sell_short };
```

### Return a structured error on fallthrough

In a server that has to stay up under bad data, fallthrough on an
exhaustive switch should produce a structured error the request handler
turns into a reject. `std::unreachable()` is the wrong tool here. It
promises the compiler the case cannot happen; reaching it is UB, which
may surface as a crash, a wrong return, or a miscompile of surrounding
code. Bad casts, memory corruption, and deserialized structs with garbage
bits all reach it.

Every fallible step -- including exhaustive switches over domain enums --
returns `lib::result<T>`. Fallthrough builds a typed error via
`lib::new_error`; the top-level `try_handle_all` (see `error-handling.md`)
translates it into a reject carrying the offending field and value.

```cpp
// GOOD - exhaustive switch; fallthrough is a structured error
auto to_internal(wire::side from) -> lib::result<side>
{
  switch (from) {
    case wire::side::buy:        return side::buy;
    case wire::side::sell:       return side::sell;
    case wire::side::sell_short: return side::sell_short;
  }

  return lib::new_error(errors::invalid_field_value{
      .field = field_tag{tags::side},
      .value = fmt::format("{}", from),
  });
}
```

The compiler still warns when a new `wire::side` enumerator is added; the
fallthrough only fires for values that bypassed the type system upstream.
At that point a typed error is the right response -- the request handler
rejects the affected message with "invalid side" rather than taking the
process down.

Boundary parsers follow the same shape. The input is a `char` or `int`
rather than an enum, so a `default:` arm is required:

```cpp
// GOOD - boundary parser: default arm required
auto side_from_fix_char(char c) -> lib::result<side>
{
  switch (c) {
    case '1': return side::buy;
    case '2': return side::sell;
    case '5': return side::sell_short;
    default:
      return lib::new_error(errors::invalid_field_value{
          .field = field_tag{tags::side},
          .value = fmt::format("'{}'", c),
      });
  }
}
```

### When `std::unreachable()` is still appropriate

Reserve it for cases where the function cannot meaningfully return
`lib::result` and the input has been validated upstream:

- a `constexpr` helper used at compile time, where bad input is a build
  error rather than a runtime condition;
- a logging or formatting helper that must produce *some* string -- and
  even there, a sentinel like `"unknown"` is preferable to UB on a
  corrupted value.

If `lib::result` seems heavy in a hot path, note that the conversion cost
lands once at the boundary handler -- internal call sites just propagate.
`assert(false)` carries the same risk: it disappears under `NDEBUG`, and
the function falls off the end into UB.

### `if/else` chains need an explicit `else`

The compiler does not exhaustiveness-check `if/else`, even over a strong
enum. Always include a final `else` with explicit handling -- a
`lib::new_error` for fallible code, an unambiguous default otherwise.

## Designated initializers

Prefer designated initializers over positional initialization for aggregate
types. Three reasons:

1. **Self-documenting.** The field name sits next to its value at the call
   site, so the reader does not need to consult the struct definition to
   decode positional arguments.
2. **Refactor-safe.** Adding a field warns at every initializer missing that
   field; renaming a field errors at every initializer that still uses the old
   name. Both are much easier to find than a silently shifted positional
   argument.
3. **Swap-resistant.** C++20 requires designators to appear in declaration
   order, so two adjacent fields of compatible types -- the easiest swap
   mistake to make positionally -- cannot be silently transposed. Out-of-order
   designators fail to compile rather than producing a working program with
   the wrong meaning.

```cpp
// BAD - positional; what do true and 1024 mean here?
auto config = server_config{"0.0.0.0", 8080, true, 1024};
```

```cpp
// GOOD - each value is labeled
auto config = server_config{
  .listen_address = "0.0.0.0",
  .listen_port = 8080,
  .reuse_address = true,
  .backlog = 1024,
};
```

**Trailing commas.** Use trailing commas on multi-line initializers. Diffs stay
one line; clang-format keeps the multi-line layout. Single-line or single-field
initializers can omit the trailing comma.

```cpp
// Short single-line is fine without a trailing comma
auto error = rejection{.reject_code = error_code::price_required};
```

**Omit defaulted fields.** Fields that already have a default initializer in
the struct definition can be omitted -- only name fields you actually want to
set. Restating defaults duplicates the source of truth.

```cpp
struct order_options {
  side                 order_side;                    // required, no default
  std::optional<price> limit_price{};                 // optional, defaulted
  time_in_force        tif = time_in_force::day;      // defaulted
};

// BAD - restates the defaults
auto opts = order_options{.order_side = side::buy, .limit_price{}, .tif = time_in_force::day};

// GOOD - only name what differs from the default
auto opts = order_options{.order_side = side::buy};
```

**Brace elision for non-scalar fields.** When a field's declared type is a
class type -- strong type, container, or similar -- prefer `.field{value}`
over `.field = some_type{value}`. The compiler already knows the declared type;
repeating it is noise. For scalar fields, use `=` for clarity
(`.port = 8080`, `.enabled = true`).

```cpp
// BAD - the type name is redundant
auto t = replace_tag{.source = tag{Tags::Account}, .target = tag{Tags::ClOrdID}};

// GOOD - brace elision; the compiler deduces the type from the field's declared type
auto t = replace_tag{.source{Tags::Account}, .target{Tags::ClOrdID}};
```

**Strong-type fields.** Brace elision is especially compact with strong
types: the field name labels the role and the compiler constructs the strong
type, so neither the type name nor the underlying primitive appears twice at
the call site.

```cpp
// GOOD - the field name labels the role; the compiler constructs the strong type
auto event = order_placed{
  .order_id  {"O-12345"},
  .request_id{"R-98765"},
};
```

The same designated-initializer guarantees -- self-documenting at the call
site, refactor-safe under field rename, swap-resistant -- compose with the
strong-type guarantees from the top of this file. A renamed field surfaces
at every call site; a new field added without a default produces a warning
wherever the struct is built; and a strong-typed field cannot silently
accept the wrong primitive.

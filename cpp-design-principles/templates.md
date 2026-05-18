# Templates: concepts and dispatch

Use concepts to name the contract a template parameter must satisfy. Use
inline `requires` to dispatch at compile time on whether an expression is
well-formed.

## Concept constraints

A concept names the operations a template parameter must support. The
constraint sits in the function signature, so the wrong type fails to
compile at the call site with a readable diagnostic.

```cpp
// Capability the template parameter must offer
template <typename T>
concept serializable = requires(const T& t, json::value& j) {
  { write_json(t, j) } -> std::same_as<void>;
};

// Concept used to constrain the function
template <serializable T>
void publish(const T& message);
```

Failures land at the call site:

```cpp
publish(42);
// error: 'int' does not satisfy 'serializable'
//   note: the required expression 'write_json(t, j)' is invalid
```

Concepts compose like predicates -- `&&`, `||`, parentheses -- so a larger
concept builds from smaller named ones.

```cpp
template <typename T>
concept timestamped_event = serializable<T> && requires(const T& t) {
  { t.timestamp } -> std::convertible_to<std::chrono::nanoseconds>;
};
```

A concept is the single definition of what `serializable` means.

## Compile-time dispatch with inline `requires`

Inside a generic function, `if constexpr (requires { ... })` lets the
compiler pick a branch based on whether an expression would compile for the
current template parameter. The pattern applies when only some variant
alternatives carry a field and a visitor extracts it where present.

The variant alternatives below use the static-constexpr discriminator
pattern described in `compile-time-correctness.md`:

```cpp
struct order_placed {
  static constexpr auto exec_type = types::exec_type::placed;
  types::request_id request_id;
  // ...
};

struct order_filled {
  static constexpr auto exec_type = types::exec_type::filled;
  // ...
};

struct order_canceled {
  static constexpr auto exec_type = types::exec_type::canceled;
  types::request_id request_id;
  // ...
};

using order_event = std::variant<order_placed, order_filled, order_canceled>;
```

```cpp
// BAD - dispatch on alternative type; every new alternative needs an arm
auto request_id_of(const order_event& ev) -> std::optional<types::request_id>
{
  return lib::match(
    ev,
    [](const auto& alt) -> std::optional<types::request_id> {
      using T = std::decay_t<decltype(alt)>;
      if constexpr (std::is_same_v<T, order_placed>) {
        return alt.request_id;
      } else if constexpr (std::is_same_v<T, order_canceled>) {
        return alt.request_id;
      } else {
        return std::nullopt;
      }
    });
}
```

```cpp
// GOOD - dispatch on capability; the visitor asks "do you have a request_id?"
auto request_id_of(const order_event& ev) -> std::optional<types::request_id>
{
  return lib::match(
    ev,
    [](const auto& alt) -> std::optional<types::request_id> {
      if constexpr (requires { alt.request_id; }) {
        return alt.request_id;
      } else {
        return std::nullopt;
      }
    });
}
```

The inline `requires` reads as "if the alternative has a `request_id`, use
it; otherwise nothing." The dispatch follows the data, so a new variant
alternative is covered automatically.

## Dependent false for template `static_assert`s

Use a dependent `false` when a `static_assert` in a template should fail
only for the current instantiation. The common case is the final branch of
an `if constexpr` chain. A plain `static_assert(false, "...")` is checked
when the template is parsed, before `if constexpr` has discarded unreachable
branches. Making the asserted value depend on the template parameter delays
the assertion until the compiler selects that branch.

```cpp
template <typename...>
inline constexpr bool dependent_false = false;
```

```cpp
// BAD - the assertion fires while the template is parsed
template <typename Pred>
void wait_for_ready(Pred&& pred)
{
  if constexpr (std::same_as<std::remove_cvref_t<Pred>, bool>) {
    wait_until([&] { return pred; });
  } else if constexpr (std::is_invocable_r_v<bool, Pred>) {
    wait_until([&] { return pred(); });
  } else {
    static_assert(false, "Pred must be bool or invocable as bool");
  }
}
```

```cpp
// GOOD - the assertion fires only for the unsupported instantiation
template <typename Pred>
void wait_for_ready(Pred&& pred)
{
  if constexpr (std::same_as<std::remove_cvref_t<Pred>, bool>) {
    wait_until([&] { return pred; });
  } else if constexpr (std::is_invocable_r_v<bool, Pred>) {
    wait_until([&] { return pred(); });
  } else {
    static_assert(dependent_false<Pred>, "Pred must be bool or invocable as bool");
  }
}
```

Use the same pattern for intentionally disabled template instantiations:

```cpp
template <typename T>
  requires std::same_as<T, std::string>
auto parse_field(std::string_view) -> T
{
  static_assert(dependent_false<T>, "parse_field<std::string> disabled; use string_view");
}
```

For non-type template parameters, make the helper depend on values instead
of types:

```cpp
template <auto...>
inline constexpr bool dependent_false_value = false;

template <auto Mode>
void open_file()
{
  if constexpr (Mode == open_mode::read_write) {
    // ...
  } else if constexpr (Mode == open_mode::read_only) {
    // ...
  } else {
    static_assert(dependent_false_value<Mode>, "invalid open mode");
  }
}
```

Reach for `dependent_false` in the branch that represents "the caller gave
this template a shape we do not support." Use a concept instead when the
contract belongs in the function signature.

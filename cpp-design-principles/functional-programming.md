# Functional programming

Treat functions and types as the unit of design. The closer code stays
to "values in, value out, no ambient state," the less to track at
runtime and the more the compiler can check.

Where the patterns overlap with other guides, this file states the
functional framing once and points at the deeper treatment:

- Lazy pipelines, named predicates, and predicate factories live in
  `declarative-style.md`.
- `if constexpr (requires { ... })` capability dispatch lives in
  `templates.md`.
- `lib::result`, `std::expected`, and structured error vocabularies live in
  `error-handling.md`.
- Functional core / imperative shell lives in `architecture.md`.

## Pure functions and value semantics

Take inputs by value or `const&`, return a value, do not touch globals,
do not mutate, do not log. The result is independently testable,
trivially inlineable, safe to move between threads, and reusable in
contexts the original caller never anticipated.

```cpp
// BAD - reads a global, mutates an output parameter, logs on the side
int discount_pct = 0;

void apply_discount(order& o) {
  o.total -= o.total * discount_pct / 100;
  spdlog::info("discounted {} to {}", o.id, o.total);
}
```

```cpp
// GOOD - inputs in, value out, no ambient state
money apply_discount(money total, int discount_pct) {
  return total - total * discount_pct / 100;
}
```

The good version folds over a range, runs at compile time on `constexpr`
inputs, reads from a test with two literals, and reuses for a quote that
never becomes an order. The bad version does none of that.

Pure inner layers only stay pure if their inputs are trustworthy.
Parsing at the boundary (`compile-time-correctness.md` "Parse, don't
validate") produces refined types whose constructors reject anything
malformed, so downstream pure functions need no defensive re-checks.

`[[nodiscard]]` on a value-returning function is the smallest enforceable
purity signal the language offers: ignoring the result is almost always
a bug.

## Pattern matching on sum types

Consume a `std::variant` with `lib::match`, defined in
`cross-cutting.md`. The visitor becomes a table of cases the compiler
checks for exhaustiveness; an `if/else` chain on `std::holds_alternative`
does not:

```cpp
// BAD - if/else chain on std::holds_alternative
std::string describe(const event& ev) {
  if (std::holds_alternative<order_placed>(ev)) {
    const auto& p = std::get<order_placed>(ev);
    return std::format("placed {}", p.request_id);
  }
  if (std::holds_alternative<order_filled>(ev)) {
    return "filled";
  }
  if (std::holds_alternative<order_canceled>(ev)) {
    const auto& c = std::get<order_canceled>(ev);
    return std::format("canceled {}", c.request_id);
  }
  std::abort();  // missing alternatives are found at runtime
}
```

```cpp
// GOOD - one arm per alternative, exhaustiveness checked at compile time
std::string describe(const event& ev) {
  return lib::match(ev,
    [](const order_placed&   p) { return std::format("placed {}",   p.request_id); },
    [](const order_filled&    ) { return std::string{"filled"}; },
    [](const order_canceled& c) { return std::format("canceled {}", c.request_id); });
}
```

Add an alternative to `event` and `describe` fails to compile until it
gains an arm. The visitor is the contract.

Use `lib::match` when every alternative is part of the operation's
contract, when the operation returns a value, when failures propagate through
`lib::result`, or when adding a new alternative should force this call site
to be revisited. Use `lib::match_partial` only when the caller intentionally
subscribes to a few alternatives and all others are irrelevant no-ops.
Domain decisions should stay exhaustive.

For alternatives that share shape, a single generic arm with `if
constexpr (requires { ... })` handles them uniformly; see `templates.md`
"Compile-time dispatch with inline `requires`" and
`compile-time-correctness.md` for the `static constexpr` discriminator
pattern the alternatives use.

## Higher-order functions

Pass behavior as a value -- lambdas, function pointers, function
objects, type-erased wrappers. `declarative-style.md` "Named
predicates" is the home for the everyday cases: lifting inline
conditions into named lambdas, predicate factories that capture a
threshold, projections that name "by which field."

### A small in-house functional vocabulary

When the same combinator appears twice in the codebase, lift it into a
named helper. Before adding to the in-house namespace, check
`boost::hof` and `rollbear::lift`; both ship `compose`, `partial`,
`flip`, and folds. Reinventing them rarely pays.

When intermediate types or extra arguments force `bind_front` gymnastics,
prefer a named lambda with structured bindings over the combinator chain.

### Storing callables

Type-erased callable parameters let the caller plug in behavior
without inheritance. The trade-off is the indirect call and, for
`std::function`, a possible heap allocation when the target exceeds
the SBO budget.

| Wrapper                  | Owns target? | Allocates?         | Use for                                |
|--------------------------|--------------|--------------------|----------------------------------------|
| `std::function`          | yes          | possibly (SBO-dep) | long-lived callbacks, heterogeneous    |
| `std::function_ref`      | no           | no                 | parameters that do not outlive the call|
| `lib::inplace_function`  | yes          | never              | callbacks in tight-budget paths        |
| template parameter       | yes          | no                 | hot loops, header-only generic code    |

Default to `std::function_ref` (C++26; widely available as
`tl::function_ref`) for callback *parameters*: it borrows, never
allocates, accepts any callable. Reach for `std::function` only when
the wrapper outlives the call site.

```cpp
// GOOD - the parameter is a borrowed callable; no allocation, no ownership
void for_each_active(std::span<const user> users,
                     std::function_ref<void(const user&)> visit) {
  for (const auto& u : users)
    if (u.active) visit(u);
}
```

Outbound callbacks on pipeline stages are the canonical storing case;
`pipelines.md` shows the `on_*` field convention and the
size trade-off between `lib::inplace_function` and `std::function`.
`performance.md` "Type-erased callables on hot paths" covers when the
indirection cost forces a template parameter instead.

## Compile-time predicates and folds

Concepts and `if constexpr (requires { ... })` are the runtime-style
predicates lifted to types; `templates.md` is the home for both, as
template constraints and as capability dispatch inside a generic
function. The remaining piece is the fold over parameter packs:

```cpp
// GOOD - all arguments must satisfy the predicate
template <typename... Ts>
constexpr bool all_arithmetic = (std::is_arithmetic_v<Ts> && ...);

// GOOD - sum across a heterogeneous pack
template <typename... Ns>
constexpr auto sum(Ns... ns) {
  return (ns + ...);
}
```

`(args + ...)` and `(0 + ... + args)` replace the recursive-template
idiom that used to do this work. Reach for them before writing a helper
template.

## When functional style costs you

Functional style is the default. Step off it for one of three
reasons, all covered in more depth elsewhere in the guide.

**Type erasure has a runtime cost.** A wrapped callable goes through
a function pointer; a callable passed as a template parameter does
not. On a measured hot path, take the callable as a template
parameter so the call inlines. `pipelines.md` "Type erasure has a
cost" and `performance.md` "Type-erased callables on hot paths" cover
the same trade-off from the pipeline-stage and storage angles.

**Materializing a view costs an allocation.** A view is free; turning
it into a `std::vector` is not. `declarative-style.md` "Compose
lazily; materialize once" covers the cost model and when to convert.
The one gotcha worth surfacing here: if a caller iterates the view
twice, materialize it first -- `filter_view` re-runs the predicate
per traversal, and so does any pipeline built on it.

**Captures have lifetimes.** A lambda that captures by reference is a
pointer with sugar. If the closure outlives its captures -- stored in
a container, posted to an executor, returned from a function -- those
references dangle.

```cpp
// BAD - factory returns a lambda referring to a local
auto make_predicate(const std::string& prefix) {
  return [&prefix](const auto& s) { return s.starts_with(prefix); };
}
// prefix is gone the moment the caller uses the returned predicate.
```

```cpp
// GOOD - capture by value, no surprise lifetime
auto make_predicate(std::string prefix) {
  return [prefix = std::move(prefix)](const auto& s) {
    return s.starts_with(prefix);
  };
}
```

`mutable` lambdas hide state where readers do not expect to find it.
They have legitimate uses (one-shot caching, generators) but warrant
a comment naming the state and its scope; without one, prefer a
non-`mutable` lambda or a small named function object.

## See also

- `declarative-style.md` -- named predicates, predicate factories,
  projections, and the lazy-view cost model.
- `templates.md` -- concepts as constraints and inline `requires` for
  compile-time capability dispatch.
- `compile-time-correctness.md` -- parse-don't-validate at the boundary
  feeds refined types into the pure inner layers.
- `cross-cutting.md` -- the `lib::match` and `lib::match_partial` helpers.
- `architecture.md` -- functional core, imperative shell; where pure
  layers sit relative to I/O and threads.
- `pipelines.md` and `performance.md` -- the cost side of
  type-erased callables.

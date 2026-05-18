# Defense-in-depth validation

Fixing a bug at one site clears the symptom but leaves the bug class
available: any new caller that makes the same mistake will produce the
same failure, and any refactor that bypasses the fixed entry point will
reopen the wound.

**Core principle:** make the bug class structurally impossible to
reintroduce -- not "fixed at one site" but "no path that could bring
it back". A value that cannot exist also cannot be wrong. For
invariants the type system cannot express, validate at every layer the
data passes through.

## Why one validation is not enough

A single check at one layer says "we fixed *this* bug". Multiple checks
at every layer say "this *class of bug* can no longer happen". The two
are not the same. Code paths multiply over time: new callers appear, new
modes get added, mocks replace real components in tests. Each new path is
an opportunity for the original mistake to recur. A validation strategy
that relies on every caller doing the right thing will eventually fail;
one that catches the mistake wherever it appears will not.

Each check is a one-liner; the cumulative effect is an invariant enforced regardless of how data arrived.

## Types first: parse, don't validate

The cheapest check is the one that never runs because the invalid
value cannot be constructed. Turn untyped input into a strong type at
the boundary, and the rest of the program operates on values that
cannot misrepresent. The check does not need to be re-stated in Layer
1, Layer 2, or anywhere else; the type system carries the fact
forward. See "Parse, don't validate" in
`../cpp-design-principles/compile-time-correctness.md` for the canonical
treatment of this design move and the parser shape that makes it work.

The type system does not reach state-dependent invariants (whether the
market is open, whether an account holds funds) or values that re-enter
from untyped sources (bytes read back from a JSON file or shared memory
must be re-parsed). The four runtime layers below cover those cases.

## Four layers

Each layer catches a different class of failure. They are not redundant
with each other; they are different shapes of the same defensive posture.

### Layer 1: parse at the boundary

When a value enters the program from an untyped source -- a wire
protocol, a config file, a CLI argument, an IPC payload -- it arrives
as a primitive that the type system cannot constrain. Convert it into
a refined domain type at that boundary and do not let the unparsed
shape propagate further in.

```cpp
auto create_order(raw_order_params raw) -> lib::result<order>
{
  BOOST_LEAF_ASSIGN(auto symbol,   parse_symbol(raw.symbol));
  BOOST_LEAF_ASSIGN(auto quantity, parse_order_qty(raw.quantity));
  BOOST_LEAF_ASSIGN(auto side,     parse_side(raw.side));

  return order{
    .symbol{std::move(symbol)},
    .quantity = quantity,
    .side = side,
  };
}
```

Each parser is the single site that enforces its invariant. The
returned `order` carries `types::symbol`, `types::order_qty`, and
`types::side`, all valid by construction; code further inside the
program receives the refined types and calls methods on them without
re-checking.

When the value already arrives typed -- the upstream component handed
the caller a `types::symbol`, not a `std::string` -- this layer
collapses into the function signature itself. The bad call does not
compile, and no runtime code runs to catch it. That is the goal:
parsing lives at the program boundary; what would have been a runtime
check becomes a compile error. The caller gets the error at the point
they can do something about it -- not five frames deep, where the
original parameters are out of scope.

### Layer 2: business-logic validation

Verify domain rules that go beyond field shape: invariants the data must
satisfy *for this operation*, even if every individual field is valid.

```cpp
auto order_service::place_order(order const& o, account const& a)
    -> std::expected<void, error_code>
{
  if (a.balance < o.notional()) {
    return std::unexpected{error_code::insufficient_funds};
  }
  if (!universe_.contains(o.symbol)) {
    return std::unexpected{error_code::unknown_symbol};
  }
  return {};
}
```

This layer catches what entry validation cannot: combinations that are
shape-valid but business-invalid. A positive quantity is valid as a
number, but not if the account cannot pay for it.

### Layer 3: environment guards

Some operations are dangerous in specific contexts -- test mode, dry-run
mode, maintenance windows -- even if the inputs themselves are valid.
Guard those operations explicitly.

```cpp
auto exchange_gateway::send_to_exchange(order const& o)
    -> std::expected<void, error_code>
{
#ifdef BUILD_TEST_MODE
  if (o.route == route::production) {
    return std::unexpected{error_code::refused_in_test_mode};
  }
#endif

  if (config_.dry_run) {
    log_info("dry run: would send order {}", o.id);
    return {};
  }
  return transport_.send(o);
}
```

These guards are not about the data; they are about the environment the
code is running in. A test should never be one bug away from talking to
production. A maintenance window should never silently pass through.

### Layer 4: debug instrumentation

The earlier layers prevent the bug. This layer captures evidence when
something does slip through, so the next debugging session starts with
context rather than from zero.

```cpp
auto execute_order(order const& o)
    -> std::expected<fill, error_code>
{
  log_debug("execute_order: id={}, symbol={}, side={}, qty={}",
            o.id, o.symbol, o.side, o.quantity);

  auto result = do_execute(o);
  if (!result) {
    log_error("execute_order failed: id={}, error={}",
              o.id, result.error());
  }
  return result;
}
```

Debug logging at the right granularity is invisible until you need it, at
which point it is the difference between a 10-minute diagnosis and a
multi-day investigation. The information it captures is cheap on the
happy path; it only matters when the happy path fails.

## A worked example

A configuration loader was returning entries with empty `name` fields,
which the registry rejected far downstream with a cryptic "internal
error". The trace ended at the loader, where the `name` field was
default-initialized and never assigned.

The single-site fix would be to populate `name` in the loader. A
defense-in-depth fix starts further out: change the type so a missing
or malformed name does not compile, then layer runtime checks for the
invariants the type cannot express.

```cpp
// The entry type encodes the shape invariant: types::entry_name has
// no default constructor, and its factory rejects empty and malformed
// input. An entry whose name is empty or invalid cannot be
// constructed.
struct entry {
  types::entry_name name;
  types::source     source;
};

// Layer 1: parsing happens inside parse_file. Either every parsed
// entry has a valid name or the parse fails and returns a typed error.
auto load_entries(std::filesystem::path path)
    -> std::expected<std::vector<entry>, error_code>;

// Layers 2-4 in the registry cover the state-dependent and
// context-dependent invariants the type cannot express.
auto registry::add(entry e) -> std::expected<void, error_code>
{
  // Logging runs first so it captures attempts that subsequent layers reject.
  log_debug("registry::add: name={}, source={}", e.name, e.source);  // Layer 4

#ifdef BUILD_TEST_MODE
  if (!e.name.starts_with("test_")) {                                // Layer 3
    return std::unexpected{error_code::non_test_name_in_test_mode};
  }
#endif

  if (entries_.contains(e.name)) {                                   // Layer 2
    return std::unexpected{error_code::duplicate_entry};
  }
  // ...
}
```

The bug class is impossible to recreate. A caller that forgets the
name field fails to compile -- there is no default value to fall back
to. A loader that produces a malformed name fails to construct the
`types::entry_name` and surfaces the parse error through Layer 1. A
test that accidentally references a production-style name hits Layer
3. The debug log in Layer 4 makes any new failure mode diagnosable
without further reproduction.

## Applying the pattern

After tracing a bug to its source (see `root-cause-tracing.md`):

1. Map the data flow. List every checkpoint the value passes through,
   from external entry to internal use.
2. Add validation at each layer, with that layer's purpose in mind. Do
   not replicate the same check across layers; let each one enforce what
   it owns.
3. Add debug logging where the failure would otherwise be silent.
4. Test each layer. Force a value that bypasses Layer 1 (for example, by
   unit-testing Layer 2 directly with a constructed input) and verify
   Layer 2 catches it.

## Where validation belongs, and where it does not

Adding checks everywhere is not free. They run on the hot path, they
show up in profiles, and they require future maintainers to keep them
consistent. A few rules keep the cost bounded.

**Encode invariants in types first.** A `types::entry_name` whose
factory rejects empty and malformed input removes the "is the name
valid" check from every site that uses one; the compiler enforces the
property and the runtime does not have to. The strong-typing,
designated-initializer, and `std::expected`-factory rules in
`../cpp-design-principles/compile-time-correctness.md`, combined with the
structured errors in `../cpp-design-principles/error-handling.md`, move
many runtime checks to compile time and to a single parse site.

**Validate at component boundaries, not on every internal function
call.** The boundary is where untrusted (or potentially incorrect)
data enters; once it is past, internal calls can assume the invariant
holds. A function that is only ever called from one place inside the
same component does not need the same defensive posture as a public
entry point.

**Use assertions for "this cannot happen", returns for "the caller
passed something invalid".** An assertion documents a programmer error
that should never occur in correct code. A returned error documents an
input error that some caller may legitimately produce. They are not
interchangeable: a `MY_ASSERT(cond)` that fires in production tells
the user the program is broken; an `error_code::invalid_input`
returned to the caller tells them they passed bad data. Pick the one
that matches what the failure actually means.

**Some invariants cannot live in types.** Whether the market is open,
whether an account has funds, whether the session is authenticated --
these depend on world state, not on the shape of the value, so they
live as runtime checks at Layers 2 and 3. Relational invariants
across several values -- "these three points are non-collinear", "the
sum of allocations does not exceed the budget" -- fall in the same
bucket. C++26 contracts (`pre`, `post`, `contract_assert`) give these
checks a uniform shape; until then, plain returns and assertions
cover the same ground.

## See also

- `root-cause-tracing.md` -- finding the original site of the bug
  whose class this guide makes structurally impossible to reintroduce.
- `../cpp-design-principles/compile-time-correctness.md` -- strong typing,
  parse-don't-validate, and designated initializers as the primary
  compile-time enforcement mechanisms for shape invariants.
- `../cpp-design-principles/error-handling.md` -- `lib::result`,
  `std::expected`, and structured error payloads.
- `../cpp-design-principles/invariants.md` -- strong-guarantee rollback
  with scope guards and commit-at-end, for the case where a layer
  rejects a value after partial state has been mutated.
- `../cpp-design-principles/architecture.md` -- domain boundaries, which
  is where parsing belongs.

## References

- Alexis King, *Parse, don't validate* (2019):
  https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/
- Hillel Wayne, *Introduction to Contract Programming*: types vs
  contracts, and the relational invariants types cannot express:
  https://www.hillelwayne.com/post/contracts/
- C++26 Contracts (P2900): runtime preconditions, postconditions,
  and assertions as a first-class feature:
  https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2025/p2900r14.pdf

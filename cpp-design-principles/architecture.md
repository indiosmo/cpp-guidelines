# Architecture

How the system is structured at the largest scale: where domain boundaries
fall, which way dependency arrows point, and where side effects live.

## Theory of the domain

Source code is the precipitate of a theory the team holds about the domain;
keeping the code malleable is how that theory keeps refining itself. Naur's
1985 essay frames a program as a theory living in the heads of the developers
who built it. The theory covers the mapping from world affairs to program
text, the justifications for the chosen structure, and the judgment about
which proposed extensions fit and which strain the design. The source files
are an artifact of that theory; they do not contain it.

Domain context saturates the code. A `client_order_id` is not a string; it is
a string carrying a specific lifecycle, ownership, and uniqueness contract
the team agreed on, and that contract is what makes the surrounding code make
sense. Strong types, explicit signatures, parse-don't-validate at the
boundary, scope-guarded rollback -- the rules in the rest of this guide --
exist to keep the code legible to the theory and pliable as the theory
sharpens. Production use is where the theory meets reality, and the code has
to bend to absorb what the team learns. A rigid codebase forces the theory
to fossilize around yesterday's understanding; a malleable one lets each
sharpening of the theory land as a refactor rather than a workaround. This
is Brooks' essential complexity surfacing in the medium Reeves identified:
the source is the design, and the design is where the domain theory lives.

Documentation cannot transmit the full theory -- some of it is tacit, held in
the reflexes of the people who shipped the system -- but it is the most
durable channel for the why that code alone cannot show. Treat this as a
working lens, not a closed proof. Original authors are not automatically the
best theorists of the current system; theory can decay into folklore that
nobody can justify; and Naur's framing is sometimes invoked to excuse thin
documentation, which only accelerates the decay. Domain-Driven Design's
ubiquitous language is the practical mechanism that keeps the theory legible
in the code: the words on the whiteboard, in the tickets, in the types, and
in the tests are the same words, so a change in understanding has a single
vocabulary to travel through.

### References

- Naur, "Programming as Theory Building," 1985. Reprinted in *Computing: A
  Human Activity* (1992). PDF:
  https://pages.cs.wisc.edu/~remzi/Naur.pdf
- Brooks, "No Silver Bullet," 1986.
  https://worrydream.com/refs/Brooks_1986_-_No_Silver_Bullet.pdf
- Reeves, "What Is Software Design?," *C++ Journal*, 1992.
  http://www.developerdotstar.com/mag/articles/reeves_design.html
- Evans, *Domain-Driven Design*, 2003 (Ubiquitous Language).

## Domain separation with adapters

Components have clearly defined domains, and they do not depend on things
outside their domain. Cross-domain communication happens through dedicated
adapter modules whose only job is to translate between two neighboring
domains.

A pure domain stays testable in isolation, because
nothing in it reaches out to a vendor SDK or a sibling domain that would drag
in unrelated concerns. And when an integration changes -- a new vendor, a new
wire protocol, a new normalized concept -- the change has a single home rather
than spreading through every component that touched the old shape.

A typical layered example -- an order routing stack reaching an exchange
through a FIX vendor SDK, in the order a dependency arrow would point:

```
normalized_routing   Normalized order routing concepts, protocol-agnostic
  |
fix_routing          FIX expression of normalized_routing, vendor-agnostic
  |
fix_vendor           Adapter: glues fix_routing to vendor_sdk
  |
vendor_sdk           Vendor library wrapper; exposes the SDK's own types.
```

`normalized_routing` stays protocol-agnostic; `fix_routing` stays vendor-agnostic;
`fix_vendor` is the only place that has to change when the vendor SDK is
swapped. To add a second wire protocol (a binary order-entry protocol, say),
add a sibling `binary_routing` plus its own `binary_vendor` adapter. The
caller built on `normalized_routing` is unaffected.

A side integration follows the same shape. If `normalized_routing` needs to
talk to an unrelated `risk` domain, the integration lives in a
`normalized_routing_risk_adapter` module that depends on both -- not in
`normalized_routing` or `risk`.

### Domain-owned types

A domain owns its types, even when a neighboring domain defines something that
looks identical. `fix_routing` may have its own `order_id` that happens to be
the same `std::string` as `normalized_routing::order_id` -- they are still
distinct types, defined in their own namespaces, converted at the adapter
boundary.

Each domain can change its own type
-- widen the underlying primitive, add a tag, replace a `std::string` with a
`lib::fixed_string<32>` -- without rippling changes through every dependent. The
adapter absorbs the change in one place. If two domains keep needing to
translate each other's representations, that is a signal that an adapter is
missing, not that the types should be merged.

The alternative -- a "shared types" library that both sides reach into --
silently couples them. A change to suit one domain becomes a breaking change
for the other; a refactor blocked by a sibling's release schedule is a
refactor that will not happen. Domain-local types keep each side
independently evolvable, and the adapter layer is the place where temporary
shims can absorb a phased migration without leaking it into either domain.

See `compile-time-correctness.md` for how individual types are defined inside a domain.

Composition layers built on top of multiple domains are expected; an application
that wires order routing, a risk engine, and external APIs through their
respective adapters is one example. Such a layer depends on each domain it
composes, translates between their types, and exposes its own surface upward.

## Forward-only dependencies

Dependencies form a DAG: arrows point one way.
This rule applies at every level: between libraries, between translation units
inside a library, between headers, and between functions in a call graph. A
reader following the arrows should never have to circle back.

The point is local reasoning. To understand `B`, you read `A` first; to
understand `A`, you do not need to know what `B` does. With cycles, neither
end is the entry point -- you have to hold both in your head simultaneously
to make sense of either.

When two components both need shared functionality, extract a third
component they both depend on. Do not let `A` and `B` know about each
other directly.

```
A <-> B           A   B
                   \ /
                    C
 (cycle)         (shared)
```

CMake targets, headers, and translation units obey the same rule. A header
should never include a header that includes it back -- not even transitively.
If two libraries find themselves wanting to depend on each other, the shared
piece belongs in a third library both depend on.

### No implicit contracts

Dependency direction governs data flow inside the call graph as well. A function
should take everything it needs as an argument and return everything it
produces. It should not read state that a previous call left behind, and a
callee should not leave behind state for the caller to pick up.

The exception is a member function whose role is to mutate the class -- and
even then, the name should make the mutation obvious (`insert(value)`,
`clear()`, `reset_to(state)`). The same rule extends to free functions that
mutate an argument for performance reasons: pass by non-const reference and
use a verb form that signals mutation (`set_defaults(data&)`,
`reset(data&)`, `normalize_in_place(data&)`). What breaks the call graph is
not mutation itself, but mutation hidden behind a signature that looks pure.
A free function that "outputs" a field on a context struct the caller is
expected to read after the call makes the call graph impossible to follow
without simulating the program in your head.

```cpp
// BAD - implicit contract: rank() depends on a column that score() expects
//       untie() to have left behind on the working set
struct working_set {
  table data;
  std::optional<std::string> tiebreaker_column;  // set by untie(), read by rank()
};

void untie(working_set& ws)
{
  ws.tiebreaker_column = build_tiebreaker(ws.data);  // mutates ws.data too
}

void rank(working_set& ws)
{
  // only works if untie() was called first
  apply_monotonic(ws.data, *ws.tiebreaker_column);
  ws.data.drop_column(*ws.tiebreaker_column);
  ws.tiebreaker_column.reset();
}

void score(working_set& ws)
{
  untie(ws);
  rank(ws);
}
```

```cpp
// GOOD - each function takes what it needs and returns what it produces
auto untie(const table& data) -> column
{
  return build_tiebreaker(data);
}

auto rank(const table& data, const column& tiebreaker) -> table
{
  return apply_monotonic(data, tiebreaker);
}

auto score(const table& data) -> table
{
  return rank(data, untie(data));
}
```

The good version reads top-down: `score` needs ranking, ranking needs a
tiebreaker, the tiebreaker comes from `untie`. The bad version requires the
reader to know that `untie` leaves a residue on `working_set` that `rank`
will consume, that the consumption is destructive, and that the caller's
contract includes calling them in this exact order. None of that is visible
from the function signatures.

## Functional core, imperative shell

*Push effects towards the edge*.

Each component separates pure functional logic from runtime concerns
(threads, I/O, sockets, timers, anything that talks to the outside world).
The functional layer is what you can unit-test without spinning up a thread
pool; the runtime layer is the composition root that wires the functional
pieces to the operating system. The pattern goes by a few names --
functional core / imperative shell, hexagonal architecture, ports and
adapters -- but the rule is the same: keep effects at the edge, not in the
middle.

```
component/             Functional: message handling, state transitions, validation
component/runtime/     Runtime: thread composition, I/O, timers
```

The runtime layer depends on the functional layer, never the other way
around. The functional layer takes its inputs by reference or value,
returns results synchronously, and is exercised in tests by feeding it
crafted inputs and inspecting the outputs. The runtime layer's job is to
call those functions on the right thread at the right time, and to bridge
their results to the sockets, files, and timers the operating system
provides.

A practical consequence is that **I/O lives near `main`**. A library deep in
the dependency graph should not open a socket, read a configuration file,
or call out to a logging backend on its own initiative; it should produce
results and let the layers above route them. Pushing effects out to the
edge keeps the inner layers reusable across runtimes, trivial to test
without mocks, and makes the program's interactions with the outside world
enumerable in one place rather than scattered through the call graph.

Threading is the effect this rule treats most carefully. A
synchronization primitive (atomic, mutex, condition variable) inside an
inner component is the same kind of leak as a socket inside a domain
class. The runtime layer holds threads; inner layers run on whatever
thread the runtime hands them, with no awareness of which one. The
data-flow analogue lives in `pipelines.md`: each stage's `send` runs
synchronously, and the wiring lambda assigned to each `on_*` callback is
where the choice of consumer thread (or no thread switch at all) lives.
`runtime.md` covers the primitives.

The same separation pays off when porting to a different runtime (a new
event loop, a different threading model, a simulator for backtesting): the
functional layer moves over unchanged, and only the shell is rewritten.

The local composition rules in `declarative-style.md` describe how an inner
function reads top-to-bottom; this separation is the same idea one level
up, applied to the whole component.

## Testability without mocks

A useful check on the design above is what a unit test needs to exercise a
piece of it. If the test can hand the code plain values and inspect plain
return values, effects have been pushed out, dependencies are explicit, and
the domain is not silently reaching at anything outside itself. If the test
instead needs a mock vendor SDK, a fake thread pool, or a large fixture
struct just to make the code run, one of the preceding rules has been bent
-- an effect leaked inward, a dependency arrow points the wrong way, or
hidden state carries a contract.

Integration tests are a separate matter: they exist precisely to exercise
the wiring at the edge, and they legitimately need real sockets, real
threads, and realistic fixtures. The litmus test applies to unit tests of
the inner layers, where heavyweight setup is a signal to look at the design
rather than to invest in better mocks.

See `../cpp-testing-principles/philosophy.md` for the corresponding view
from the testing side: tests derive expected values from the domain rather
than the implementation, and a unit test of an inner layer should not
need to know about threads, I/O, or sibling components.

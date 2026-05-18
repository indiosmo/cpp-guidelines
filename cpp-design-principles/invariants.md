# Invariants, transactions, and rollback

A function that performs several mutations on shared state -- inserting into
two containers, updating an index, calling a fallible step in between -- can
leave its target in an inconsistent state if a middle step fails. Two
strategies keep such functions all-or-nothing, and idempotency matters when
steps may be retried.

The vocabulary here is the standard exception-safety taxonomy:

- **Basic guarantee.** No leaks, no UB; the object remains usable but may
  hold partially-updated state.
- **Strong guarantee.** On failure, the object is left exactly as it was
  before the call. The operation either fully applies or has no effect.
- **No-throw guarantee.** The operation cannot fail. Reserved for things
  like swap, move, and small primitive operations.

Mutating methods on classes with non-trivial invariants should target the strong guarantee.

## Strategy 1: commit at the end

When the new state can be built in locals, build it there first, then move
it into the members in one step. Every fallible operation happens against
the local; failure throws away the local and never touches the object.

```cpp
// BAD - half-applied state on failure
struct subscription_set {
  boost::unordered_flat_map<topic_id, subscriber> by_topic_;
  boost::unordered_flat_set<topic_id>             pending_;

  auto add(topic_id id, subscriber s) -> lib::result<void> {
    by_topic_.emplace(id, std::move(s));            // (1) applied
    BOOST_LEAF_CHECK(s.connect());                  // (2) may fail -- (1) stays applied
    pending_.insert(id);                            // (3)
    return {};
  }
};
```

```cpp
// GOOD - all fallible work runs on locals; the swap is the only mutation
struct subscription_set {
  boost::unordered_flat_map<topic_id, subscriber> by_topic_;
  boost::unordered_flat_set<topic_id>             pending_;

  auto add(topic_id id, subscriber s) -> lib::result<void> {
    BOOST_LEAF_CHECK(s.connect());                  // fail here: members untouched
    by_topic_.emplace(id, std::move(s));            // both inserts are no-throw at this point
    pending_.insert(id);                            // (reserve()'d at startup; see performance.md)
    return {};
  }
};
```

This is the cheapest pattern when it fits: rollback is implicit via destructors, so every new fallible step inherits rollbacks for free.

## Strategy 2: scope-guard rollback

Commit-at-end stops working when the mutations have to be visible to the
fallible step itself -- typically when one of the fallible steps needs to
*see* an earlier mutation (e.g. a downstream check reads the container we
just inserted into). The fix is to apply each mutation eagerly, but arm a
scope guard that undoes it; only on success do we `dismiss()` the guard.

A scope guard is a small RAII helper that runs a callback at scope exit
unless `dismiss()` is called first. Common shapes include `boost::scope::scope_exit` and `gsl::finally`; in-house macros work the same way.

```cpp
// BAD - second step fails, first insertion silently sticks around
auto risk_engine::add_request(const request& r) -> lib::result<void> {
  orders_.emplace(r.order_id, build_order_state(r));         // (1) applied
  requests_.emplace(r.request_id, build_request_state(r));   // (2) applied
  BOOST_LEAF_CHECK(limits_.try_add(r));                      // (3) may fail
  return {};
}
```

```cpp
// GOOD - each mutation is paired with a rollback that fires unless
//        we reach the dismiss() at the bottom
auto risk_engine::add_request(const request& r) -> lib::result<void> {
  orders_.emplace(r.order_id, build_order_state(r));
  auto undo_order = lib::scope_exit([&] { orders_.erase(r.order_id); });

  requests_.emplace(r.request_id, build_request_state(r));
  auto undo_request = lib::scope_exit([&] { requests_.erase(r.request_id); });

  BOOST_LEAF_CHECK(limits_.try_add(r));   // failure: both undos fire

  undo_request.dismiss();
  undo_order.dismiss();
  return {};
}
```

The guards fire on every exit path -- the LEAF error returns, exceptions
from `try_add`, an early `return` added later. The only path that skips
them is the success path that explicitly dismisses them.

**Guard order.** Stack-allocated objects destruct in reverse declaration
order, so the *last* mutation is undone first. That ordering usually
matches what you want (LIFO unwind), but make sure each rollback can run
without the state the later mutations introduced -- otherwise the unwind
itself becomes order-dependent. In the example above, erasing from
`requests_` and erasing from `orders_` are independent, so either order
works.

**Composing across helpers.** When a multi-step operation spans several
private helpers, each helper can return a guard to its caller. The caller
chains them and only dismisses them once the whole operation is committed.

```cpp
auto lifecycle_tracker::process(const request& r)
    -> lib::result<std::pair<request_node, order_node>>
{
  BOOST_LEAF_ASSIGN(const auto& request_node, requests_.process(r));
  auto undo_request = lib::scope_exit([&] { requests_.send_failed(r); });

  BOOST_LEAF_ASSIGN(const auto& order_node, orders_.process(r));

  undo_request.dismiss();
  return std::pair{request_node, order_node};
}
```

If `orders_.process` fails, `requests_` is told to mark the request failed
-- the same rollback path it would walk if it had hit the failure itself.
The two subsystems stay consistent without `lifecycle_tracker` having to
know their internal state.

### Commit-at-end vs scope guards

Prefer commit-at-end when you can, and reach for scope guards when you
cannot. The signal that you cannot is usually one of:

- A later fallible step needs to *observe* the earlier mutation (e.g. it
  walks `orders_` and the new entry has to be visible to it).
- The mutation has externally observable effects (a notification was sent,
  a file was created, a counter was incremented elsewhere) that cannot be
  staged in a local.
- The container or subsystem does not support staging cheaply (an in-place
  hash map vs. building a new one).

Commit-at-end is cheaper, simpler, and harder to misuse; only step up to
scope guards when the work genuinely needs to be visible mid-flight.

## Idempotency

Operations get retried for many reasons: a message redelivered after a
reconnect, a rejection handled twice because two paths converged on it, or a
scheduled task firing after the work already happened. An *idempotent*
operation produces the same final state whether it runs once or many times.
Two practical patterns:

**Pre-compute the rollback amount; do not recompute on the retry.** When
the same operation may be reversed later (a risk check that reserves
capacity, then frees it on rejection), record what you reserved at the
time, and undo exactly that. Recomputing the amount at reversal time risks
freeing more (or less) than was originally taken, especially if the inputs
have drifted.

```cpp
// BAD - rollback recomputes from current order state; if the state has
//       changed (a partial fill arrived), we free the wrong amount.
auto on_reject(const request_id& id) -> lib::result<void> {
  BOOST_LEAF_ASSIGN(const auto& r, find_request(id));
  BOOST_LEAF_CHECK(limits_.reduce({.quantity = r.live_quantity(),
                                   .notional = r.live_notional()}));
  requests_.erase(id);
  return {};
}
```

```cpp
// GOOD - the reservation captured exactly what was reserved; the rollback
//        applies that, regardless of what else has happened since.
auto on_reject(const request_id& id) -> lib::result<void> {
  BOOST_LEAF_ASSIGN(const auto& r, find_request(id));
  BOOST_LEAF_CHECK(limits_.reduce({.quantity = r.rollback_quantity,
                                   .notional = r.rollback_notional}));
  requests_.erase(id);
  return {};
}
```

`rollback_quantity` and `rollback_notional` are written when the request
is admitted, alongside the fields they reverse. A second `on_reject` for
the same `id` returns `request_not_found`; `find_request` fails before any
limits are touched, so the state is unchanged.

**Check-then-act, atomically.** A retry-safe handler should be able to
inspect the current state and decide that the work has already been done.
`try_emplace_unique` (insert iff absent) and "find or insert" primitives
fold the check and the insert into one indivisible step, eliminating the `contains`+`insert` window that can race or duplicate work.

```cpp
// BAD - second handler call inserts a duplicate
if (!orders_.contains(id)) orders_.emplace(id, build_order(...));
```

```cpp
// GOOD - the second call observes the existing entry and does nothing
auto [it, inserted] = orders_.try_emplace(id, build_order(...));
if (!inserted) { /* already processed; no-op, or update if desired */ }
```

## Invariants belong on the type that owns them

Keep invariant-maintaining methods on the class, with the fields they
protect private. Two members that must move in lockstep are one member -- a
small struct whose methods enforce that relationship. The public API exposes
only invariant-respecting states: either a mutation commits fully or it
never happened. The scope-guard and commit-at-end patterns above are how a
method delivers on that promise; see `compile-time-correctness.md` and `architecture.md` for
the broader class-design consequences.

## See also

- `error-handling.md` -- how `lib::result` and `std::expected` express
  failure; the rollback patterns here compose with both.
- `performance.md` -- the "reserve at startup, never grow" rule turns
  `vector::push_back` and `unordered_map::emplace` into reliably
  no-throw operations, which is what makes the commit-at-end pattern
  work without a guard on the commit phase itself.
- `architecture.md` -- overall module and ownership boundaries that shape
  which types carry which invariants.

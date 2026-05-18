# Performance

## Philosophy

### Default to clarity

Performance follows from good design more often than not. Strong types,
declarative pipelines, and pushing effects to the edge usually produce code
that performs within the bounds the system actually requires. The cases where
they do not are specific, identifiable, and almost always on a measured hot
path.

### Have performance requirements before optimizing

Optimization needs a target: a latency budget, a throughput rate, a
tail-latency bound. Without one, "is this fast enough" has no answer. Cite
the requirement, then measure against it.

### Measure, do not guess

Hardware is full of surprises -- cache lines, branch predictors, prefetchers,
NUMA effects. A change that looks faster on paper can be slower in practice.
Use a benchmark harness for runtime measurements; use
https://godbolt.org to compare generated machine code when reasoning about
code-gen.

### Keep performance characteristics in mind, but do not optimize blind

Allocations, cache locality, and algorithmic complexity belong in the back of
the mind whenever code is written. The discipline is to *notice* them, not to
pre-emptively contort the code around them. A `views::transform` chain is the
right default; reaching for a hand-rolled loop should follow a measurement
that says the chain is the bottleneck.

### Evaluate in full context

Local speedups only matter in proportion to the system they live in. Spending
weeks on a 2x speedup in a component that is not on the hot path leaves a
maintenance cost behind for no end-to-end gain. Optimizing a pipeline stage
from 50 ns to 25 ns when the end-to-end latency is 50 ms saves 25 ns in a
50 ms budget: one part in two million, far below the noise floor of the
surrounding system. Pick targets where the saved time is visible against the
budget that matters.

### Declarative style and measured hot paths

Declarative composition is the default; see `declarative-style.md` for the
full treatment. The rules in the "Hot-path allocations" and "Map and ordered
container choice" sections below override that default only on a measured hot
path with a known budget being exceeded. Outside those sections, prefer the
declarative form.

### Type-erased callables on hot paths

When a type-erased callable is stored as a member on a hot path, prefer a
fixed-capacity inplace function (the SG14 `inplace_function<Sig, Capacity>`
design) over `std::function`. `std::function` heap-allocates when the
captured state exceeds its small-buffer-optimization threshold; an inplace
function stores the callable in fixed in-class storage and fails to compile
if it does not fit. The size constraint becomes a compile-time check rather
than a silent allocation at construction.

## Hot-path allocations

For hot paths, avoid heap allocations. These rules apply to identified hot paths; default to clarity elsewhere. The table below lists the usual swaps.

| Instead of                 | Use                                     | Notes                                          |
|----------------------------|-----------------------------------------|------------------------------------------------|
| `std::string`              | `lib::fixed_string<N>`              | Stack-resident; no allocation per value        |
| `std::vector<T>` (bounded) | `boost::container::static_vector<T, N>` | Stack-resident; capacity known at compile time |
| `std::vector<T>` (growing) | `std::vector<T>` with `reserve()`       | Pre-allocate at startup; never resize on the hot path |

`lib::fixed_string<N>` wraps `boost::static_string<N>` (or a small in-house equivalent modeled on the in-development `std::inplace_string<N>` proposal) to give a non-allocating, stack-resident string with a compile-time maximum length.

```cpp
// BAD - allocates on every call
std::string client_order_id = generate_client_order_id();
```

```cpp
// GOOD - fixed-capacity string, stack resident
lib::fixed_string<36> client_order_id = generate_client_order_id();
```

```cpp
// BAD - reallocates as it grows
std::vector<field> passthrough_fields;
```

```cpp
// GOOD - known upper bound, stack allocated
boost::container::static_vector<field, 12> passthrough_fields;
```

Reserve once at startup so the vector never reallocates on the hot path:

```cpp
// GOOD - variable size, but allocation happens once
std::vector<order> orders_;
orders_.reserve(config.orders_reserve_size);
```

Allocate freely at startup, in configuration loading, and in one-shot utilities. The hot path should avoid allocations.

## Map and ordered container choice

Prefer `boost::unordered_flat_map` and `boost::unordered_node_map` over
`std::unordered_map`: the Boost implementations are measurably faster and
lay out memory better. The same flat-vs-node trade-off applies to ordered
containers (`boost::container::flat_map` vs `std::map`). The choice turns
on speed versus reference stability:

- **Flat variants** (`boost::unordered_flat_map`,
  `boost::container::flat_map`) store entries in a contiguous buffer.
  Lookups and iteration are fast and cache-friendly, but insertion can
  relocate every existing entry -- by rehashing in the unordered case, or
  by shifting elements to keep sorted order in the ordered case. Any
  reference, pointer, or iterator into the map can be invalidated by the
  next insert. Use them for lookup tables populated at startup or rarely
  mutated, where no caller keeps a pointer or iterator between inserts.
- **Node variants** (`boost::unordered_node_map`, `std::map`) store each
  entry in its own allocation. Each insert carries an allocation cost and
  worse cache behavior, but references and pointers stay valid across
  insert and erase. Reach for them when the map sees frequent
  insert/delete on the hot path, or when other code needs to hold
  references into the map between operations.

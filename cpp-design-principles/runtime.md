# Runtime and inter-thread communication

Threading is an effect; this layer is where it lives. Inner components --
the stages described in `pipelines.md`, the per-domain functional layers
elsewhere -- hold no atomics, no mutexes, no condition variables. The
runtime gives each component a thread, and the component runs
single-threaded inside it. Work that crosses thread boundaries goes
through a queue. Locks are the last resort, used only when the design has
already pushed back on the alternatives.

This is the same edge-effect principle `architecture.md`'s "Functional
core, imperative shell" applies to sockets, files, and timers, sharpened
for concurrency. A synchronization primitive inside an inner component
is the same kind of leak as a socket inside a domain class. The runtime
holds threads; the inner layers run on whatever thread the runtime hands
them and stay unaware of which one.

## Single-threaded internals

A component owns a slice of state -- queues, indexes, counters, an FSM, a
cache. Every read and every write of that state happens on one thread:
the thread the runtime gives the component. The component's methods are
not synchronized, and the fields are plain members, not `std::atomic` or
mutex-guarded.

```cpp
class quote_book {
 public:
  void on_quote(const quote& update);
  auto best_bid() const -> price;

 private:
  // Plain members. Only the owning thread touches them.
  flat_map<symbol_id, level> levels_;
  std::uint64_t updates_seen_ = 0;
};
```

A consequence of pushing threading to the edge: this is the cheapest
concurrency model that works. No atomics, no fences, no reasoning about
reorderings. The reader of `on_quote` sees a function that mutates
`levels_` and increments a counter, and that is the whole story. The
cost of synchronization is paid once, at the boundary, instead of
inside every method.

The pattern only holds if cross-thread access goes through the runtime
described below. A direct call from another thread into `on_quote` is a
data race; the type system will not catch it. The "Document the deviation,
not the default" section covers where the constraint needs to be discoverable.

## Marshalling between threads

Each thread runs an event loop. The loop owns a thread-safe task queue and
spends its life pulling closures off the queue and invoking them. Posting
work to another thread is a one-liner:

```cpp
b_loop.post([&b, event]{ b.handle(event); });
```

In a pipelined program, the caller is rarely raw application code -- it is
the wiring lambda assigned to some producer's `on_*` callback (see
`pipelines.md` "Interception at the wiring point"). The producer fires the
callback synchronously; the wiring lambda turns the call into a post; the
consumer runs `handle` on its own loop, none the wiser.

The runtime guarantees the closure runs on `b_loop`'s owning thread. From
the caller's perspective, the call is fire-and-forget: it returns
immediately, and B serves the event when its loop next dequeues. From
component B's perspective, the closure looks like any other invocation on
its own thread, so the single-threaded contract on `b.handle` holds.

This is the default tool for cross-component calls. Treat it the way you
treat a direct method call: the receiver is "wherever the runtime put
it", and the post replaces the function-call instruction. The shape
stays top-to-bottom readable: A produces an event, the wiring posts it to
B, B handles it. No shared mutable state, no locks, no condition
variables.

A few practical rules:

- **Capture by value, not by reference, unless the lifetime is obvious.**
  A reference into A's frame outlives the post only if A guarantees it.
  Local copies, shared pointers, or moves into the closure are the safe
  defaults.
- **Do not block the posting thread waiting for the result.** A round-trip
  reply is another post, going the other way. Blocking turns the runtime
  into a synchronous RPC and reintroduces the deadlocks the design avoids.
- **Keep the closure small.** The captured state lives inside the task
  queue's storage; large captures force allocation. See the next section.

## The marshalling primitive

The `event_loop` exposes a narrow interface:

```cpp
class event_loop {
 public:
  using task_t = lib::inplace_function<void(), 2048>;

  void post(task_t task);   // thread-safe; called from any thread.
  void run();               // dequeues and invokes; called from the
                            // owning thread.
};
```

Two design points carry most of the weight:

**The queue is a high-quality concurrent queue.** TBB's
`concurrent_queue`, moodycamel's `ConcurrentQueue`, or any well-tested MPSC
queue is a reasonable choice. Building one is not. The contention behavior
of a hand-rolled queue is the kind of thing that looks fine in micro
benchmarks and falls over under production load.

**Tasks are stored as fixed-capacity inplace functions.** Posting must not
allocate, because allocation on a hot path turns a posted event into a
malloc-bound operation under load. `lib::inplace_function<void(), 2048>`
keeps the captured state inside the function object's own storage and
fails to compile when a closure exceeds 2048 bytes. The capacity is
generous on purpose: the cliff is loud and easy to debug, where a silent
fallback to heap allocation is not. See `performance.md` for the broader
rationale on inplace functions.

```cpp
// Posts compile and run with no heap traffic.
b_thread.post([id, payload]{ b.handle(id, payload); });

// Captures a 4 KB struct by value: the inplace_function size_t check
// fails at compile time, prompting a redesign (capture a handle, move
// a unique_ptr, or split the work into smaller posts).
b_thread.post([big_payload]{ b.handle(big_payload); });
```

The capacity is a deliberate constraint, not a quota to game. A closure
that does not fit is a closure that would have allocated; the compile
error is the design feedback.

## Per-publisher SPSC queues for high-volume streams

The MPSC queue inside `event_loop` is the right default but not the right
shape for every workload. Some sources fan out at firehose rates: a
market-data channel feeding a dozen strategies, a telemetry pipeline
emitting tens of thousands of samples a second. The contention on a single
MPSC dequeuer becomes the bottleneck long before any individual consumer
does.

For those streams, give each publisher-consumer pair its own
single-producer-single-consumer queue. Moodycamel's `ReaderWriterQueue` is
a strong default: lock-free, cache-aligned, and an order of magnitude
faster than an MPSC queue under steady load.

```cpp
// One SPSC queue per (channel, subscriber) pair.
class market_data_publisher {
 public:
  void subscribe(strategy& subscriber);
  void publish(const tick& sample);  // called from the I/O thread.

 private:
  struct subscription {
    strategy* target;
    moodycamel::ReaderWriterQueue<tick> queue;
  };
  std::vector<subscription> subscriptions_;
};
```

The trade-off is shape, not speed. SPSC queues impose a one-producer
constraint that has to be honored by construction; the runtime cannot
catch a second producer at compile time. Keep the SPSC pattern for fan-out
streams where the publisher is unambiguously single-threaded, and use the
MPSC `event_loop` everywhere else.

## Choosing between marshalling and sharing

Marshalling is the default. Lock-free primitives are an optimization for
state with the right shape. Mutexes are the last resort. The decision tree
is short:

### Default: post to the owning thread

The component has a clear owning thread, the work fits the
"event_loop.post" shape, and serializing on the owning thread does not
create a backlog. Any cross-component interaction starts here. The cost is
one queue hop and one function-pointer dispatch -- cheap, predictable, and
locally reasonable.

### Lock-free primitive: small, hot, mostly-read state

When state is small (one or a few words), trivially copyable, read often,
and written rarely, a lock-free read can be cheaper than a post. The
shapes that come up:

- **`std::atomic<T>` for flags and counters.** A `bool` "is the gateway
  connected", a `std::uint64_t` "messages sent so far". One writer, many
  readers, relaxed or acquire/release ordering depending on what the
  reader does with the value.
- **A seqlock for single-writer/multi-reader of a small struct.** A
  pricing snapshot, a configuration view, a compact set of latest values.
  The writer increments a sequence counter around the write; readers
  retry if they observe an in-flight update. No reader ever blocks the
  writer.
- **A spinlock for short critical sections under high contention.** Rare,
  and the right answer is usually "post instead". When the critical
  section is provably short and a `std::mutex` adds measurable wakeup
  cost, a spinlock can win. Always benchmark.

```cpp
// GOOD - one writer, many readers, atomic flag.
class connection {
 public:
  void mark_connected() { connected_.store(true, std::memory_order_release); }
  auto is_connected() const -> bool {
    return connected_.load(std::memory_order_acquire);
  }

 private:
  std::atomic<bool> connected_ = false;
};
```

The qualifier is "small". The moment the shared state grows past a couple
of words or wants composite invariants ("these three fields update
together"), the lock-free version becomes its own subtle concurrency
problem and the post is back on the table.

### `std::mutex` and `std::shared_mutex`: last resort

Reach for a mutex when the design has already pushed back on the
alternatives. Acceptable cases tend to share two properties: the surface
is small and clearly bounded, and posting would be the wrong shape.

- **Test utilities.** A test scaffolding object accessed from a few
  threads in a test harness, where the production code never touches it.
- **Configuration loaded once.** A shared snapshot built at startup and
  read from many threads with no further writes. A `std::shared_mutex`
  guarding the rare reload is cheaper than wiring a post for every read.
- **Slow paths.** Diagnostics, debug dumps, administrative endpoints.
  The path runs once a minute; a mutex is fine.

If a mutex appears in a hot path of a component the runtime owns, treat
it as a smell. The fix is almost always "the owning thread should hold
the state, and the rest of the world should post".

## Document the deviation, not the default

Single-threaded is the default contract for every functional component.
Restating it on every class is the same shape as adding `// pure: no side
effects` to every value-returning function: noise, drift the moment a
class is missed, and a misleading suggestion that the property is a
per-class assertion rather than a codebase convention.

Document threading where it deviates from the default:

**On a runtime-layer wrapper that owns or names threads.** An event loop,
a thread pool, a worker, a publisher with its own SPSC queue. The
wrapper exists *because* of threading; threading is part of its
interface.

```cpp
// Owns the market_data thread. Subscribers' callbacks fire on that
// thread; cross-thread access goes through post().
class market_data_publisher {
  ...
};
```

**On a class that is deliberately thread-safe.** Methods that may be
called from any thread, backed by atomics or a mutex, name the safety
property and what backs it.

```cpp
// Thread-safe: backed by an atomic. Callable from any thread.
class connection {
  auto is_connected() const -> bool;
  ...
};
```

**On a method that breaks or extends the surrounding class's contract.**
The class is otherwise single-threaded; a specific method is not.

```cpp
// Thread-safe: callable from any thread; backed by an atomic counter.
auto messages_sent() const -> std::uint64_t;
```

**On a component that runs on a non-default thread**, when a codebase
has more than one event loop. "Lives on the market_data loop, not the
api loop" is information; "lives on the loop the runtime gave it" is
not.

## Failure mode: undocumented runtime wrappers

The hazard is sharpest at the runtime layer, because that is where
threading decisions are made and where future edits are most likely to
add a wrong-thread caller. A runtime wrapper whose threading is not
documented invites one of:

- A new feature wires a callback that a different thread now invokes.
  The wrapper's methods race with the existing loop, but every call
  site looks normal.
- A test fixture spins up a worker that pokes the wrapper directly to
  drive a scenario, and the production code grows to rely on the same
  shape.
- A "small" optimization moves a method out of the loop "just for the
  read", and the read now races with a write the author did not
  consider.

Inner functional components are protected by the convention: a reader
weighing a change sees a plain class with no synchronization and knows
the runtime hands it a single thread. The runtime wrapper is where the
documented thread contract has to do its work.

## See also

- `pipelines.md` -- the per-component callback shape that runs inside
  one of these threads. The wiring lambda assigned to each `on_*`
  callback is where threading, tee, filter, and other cross-cutting
  choices land; this is where `post` calls live.
- `architecture.md` -- functional core / imperative shell. The runtime
  is the imperative shell: threads, queues, and I/O at the edge, with
  pure logic on the inside.
- `state-machines.md` -- the deferred-event pattern is the local
  analogue of `post` for re-entrance inside a single call stack.
- `performance.md` -- inplace function rationale, hot-path allocation
  rules, and map and container choice on hot paths.

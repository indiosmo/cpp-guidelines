# Pipelines

Many systems are shaped the same way: data comes in, flows through a chain
of components, and produces data going out. Each component is a class with
inbound member functions and outbound callback fields. No component knows
who feeds it or who consumes its output; a wiring layer near `main`
connects them.

The shape is the same whether the data is called events, requests,
messages, commands, or ticks. This page uses *event* as shorthand for any
of them.

Each outbound callback is named, typed, and addressed to one specific
consumer (chosen by the wiring layer). That is what keeps the pattern from
collapsing into a global message bus where every subscriber filters every
event at the receiver. The wiring graph stays a DAG of typed connections,
in line with `architecture.md`'s forward-only dependencies; the type
system is doing work the bus would have to do at runtime.

Cross-thread concerns live in `runtime.md`. The wiring lambda is the
bridge between the two: see "Interception at the wiring point" below.

## Pattern

A pipeline stage is a class with two surfaces:

- **Inbound member functions** -- one per request type the stage accepts.
  Conventionally named `send`, with overloads per request type, or named
  after the operation (`submit`, `cancel`, `route`).
- **Outbound callback fields** -- one per event the stage can produce.
  Conventionally named `on_<event>`, holding a type-erased callable.
  A stage may expose several `on_*` callbacks, including events that are
  not the primary transformation output: a strategy that emits orders may
  also emit "tranche updated"; an order tracker that consumes execution
  reports may emit "state changed" and "filled" alongside.

The stage knows nothing about who calls `send` or who is wired to `on_*`.
Both ends are filled in from outside.

```cpp
class order_router
{
public:
  void send(submit_request&& request);
  void send(cancel_request&& request);

  // Outbound: filled in by the wiring layer.
  lib::inplace_function<void(routed_order&&), Capacity> on_routed;
  lib::inplace_function<void(rejection&&),    Capacity> on_rejected;
};
```

`std::function<Sig>` works the same way; the inplace function variant
just avoids the heap allocation `std::function` performs when the captured
state spills past its small-buffer threshold. See `performance.md` for
sizing guidance; the `Capacity` placeholder above stands for whatever the
captured state needs.

The shape generalizes. A market-data normalizer exposes `on_book_update`,
`on_trade`. A document processor exposes `on_parsed`, `on_failed`. A
session adapter exposes `on_message`, `on_disconnected`. The stage only
talks outward through callbacks; it never reaches up the call graph to
find its consumers.

**Threading lives at the edge, not in the stage.** A stage's `send` runs
synchronously on whatever thread invoked it. The stage holds no atomics,
no mutexes, no thread-aware state. The wiring layer chooses which thread
the consumer's `send` runs on: a direct invocation runs it on the
producer's thread; a posted lambda runs it on the consumer's. Either
choice is invisible to both stages. This is `architecture.md`'s
"functional core, imperative shell" applied to the pipeline shape; see
`runtime.md` for the runtime primitives the choice rests on.

## Source, sink, and stage

Three roles emerge in any pipeline:

- A *source* has only outbound callbacks: a market-data feed handler, a
  file watcher, a timer.
- A *sink* has only inbound member functions: a journaller that writes
  events to disk, a metrics exporter.
- A *stage* has both: it consumes one or more inbound types and emits one
  or more outbound events.

These are vocabulary the team uses to talk about a class's role, not
types every component inherits from. A base class would either force
virtual dispatch or template every stage on its neighbors' types,
neither of which pays for itself given the surface is a few member
functions and callback fields.

## Wiring near `main`

The wiring layer instantiates the components and assigns each producer's
`on_*` to a lambda that calls the next consumer's `send`. This is the
only place in the program that holds the full topology.

```cpp
int main()
{
  auto config = load_config();

  auto session = vendor_session{config};
  auto router  = order_router{config.routing_table};

  // Router emits routed orders -> session.
  router.on_routed = [&session](routed_order&& order) {
    session.send(std::move(order));
  };

  // Session emits execution reports -> router.
  session.on_execution_report = [&router](execution_report&& report) {
    router.send(std::move(report));
  };

  session.on_rejected = [&router](rejection&& reason) {
    router.send(std::move(reason));
  };

  session.connect();
  run_event_loop();
}
```

`main` reads as the topology of the program: who produces what, who
consumes what. To understand the data flow, a reader does not have to
chase through the components -- the answer is in one place.

For a pipeline of more than two stages, the same shape repeats. When the
assignment boilerplate adds up, a project may grow a small
`wire(producer, consumer)` overload set per event type that hides the
lambda; the call site then reads
`wire<routed_order>(router, session);`. The mechanism stays the same.

## Interception at the wiring point

The assignment site for `on_*` is more than a connector. The wiring layer
chooses *what* callable a producer's callback holds; "call the consumer's
`send` directly" is the trivial choice, and the useful non-trivial choices
are the program's editing console for cross-cutting behavior. Threading,
telemetry, filtering, and re-entrance handling all land here, with
neither stage aware.

```cpp
// Direct: producer's thread continues into consumer.
producer.on_event = [&consumer](event&& e) {
  consumer.send(std::move(e));
};

// Thread marshalling: bridges to runtime.md. The consumer runs on its
// own loop regardless of who fired the event.
producer.on_event = [&consumer, &consumer_loop](event&& e) {
  consumer_loop.post([&consumer, e = std::move(e)]() mutable {
    consumer.send(std::move(e));
  });
};

// Tee: one producer, two consumers. The fan-out lives in main, not
// inside the producer (which would have to grow a subscriber list) or
// the primary consumer (which would have to know about telemetry).
producer.on_event = [&consumer, &telemetry](event&& e) {
  telemetry.record(e);
  consumer.send(std::move(e));
};

// Filter: drop events the consumer should not see.
producer.on_event = [&consumer, &universe](event&& e) {
  if (universe.contains(e.symbol)) {
    consumer.send(std::move(e));
  }
};

// Test substitution: capture into a probe instead of forwarding.
producer.on_event = [&captured](event&& e) {
  captured.push_back(std::move(e));
};
```

The unifying property is that **the callback assignment is itself a
combinator**. Every cross-cutting behavior -- threading, telemetry,
filtering, retry, throttle -- is a question of which lambda the wiring
layer hands to `on_event`. The pattern in `cross-cutting.md` is the same
idea for service handles (clock, logger, timer) chosen at startup; this
is the per-edge version for data flow.

The threading bullet is the bridge to `runtime.md`. Without the wiring
indirection, every stage would have to know which thread its consumers
run on -- precisely the coupling this pattern exists to remove. With it,
the choice of which loop runs the consumer lives one assignment away,
where the topology is also visible.

### Defer re-entrant callbacks instead of recursing

Some chains loop back into their own producer. An order flows outbound;
a stage in the middle decides to reject it and fires `on_rejected`. The
wiring layer routes the rejection back to an upstream stage that owns
the order's state. That stage, on receiving the rejection, may want to
emit a follow-up request -- retry on another venue, send a related
cancel. If the rejection handler does that synchronously, the new
request flows back down through the same chain while the first
rejection is still unwinding. A second rejection lands on a stage
already mid-call; iterators invalidate inside a loop the stack above
is still walking; recursion replaces a forward DAG with a cycle the
type system never sees.

The fix mirrors the **deferred event** pattern from `state-machines.md`:
the wiring layer (or the affected stage) places the re-entrant callback
on a queue and processes it after the current call stack unwinds. The
outermost `send` finishes its work undisturbed; the queued callback is
dispatched on the next pump of the loop.

```cpp
// Wrapper that serializes re-entrant traffic into the underlying stage.
class serialized_stage
{
public:
  void send(request&& r);
  void send(event&& e);

private:
  enum class phase { idle, sending, dispatching };

  template <typename Message>
  void try_dispatch(Message&& message);

  void drain_pending();

  phase phase_ = phase::idle;
  std::queue<deferred_action> pending_;
  inner_stage impl_;
};

void serialized_stage::send(request&& r)
{
  if (phase_ != phase::idle) {
    pending_.push([this, r = std::move(r)]() mutable {
      impl_.send(std::move(r));
    });
    return;
  }

  phase_ = phase::sending;
  impl_.send(std::move(r));   // may synchronously fire impl_.on_*
  phase_ = phase::idle;
  drain_pending();
}
```

`drain_pending` pulls from the queue while it is non-empty and the
phase is idle, so a callback enqueued during the drain is processed in
the same pump rather than nesting another invocation. The shape carries
over directly from a `boost::sml` machine with `sml::defer` on a
re-entrant event; the underlying mechanism -- "queue it now, run it
when we are not on the stack" -- is the same.

This is the local analogue of `runtime.md`'s `event_loop::post`. The
loop's queue defers work across threads; the re-entrance queue defers
work across the current call stack. Both fit the broader theme: when
an action would create a cycle in the call graph, post it. The wiring
layer is one place where that decision can live without contaminating
the stages on either side; the wrapper above is the other.

## External integrations

Components that wrap external systems -- vendor SDKs, sockets, file
watchers, OS event sources -- implement the foreign listener interface
internally and re-emit through `on_*` callbacks shaped in the domain's
vocabulary. The rest of the program never sees the vendor's types.

```cpp
class vendor_session
{
public:
  explicit vendor_session(config::session settings);

  void send(routed_order&& order);
  void connect();

  lib::inplace_function<void(execution_report&&), Capacity> on_execution_report;
  lib::inplace_function<void(rejection&&),        Capacity> on_rejected;
  lib::inplace_function<void(disconnected_reason),Capacity> on_disconnected;

private:
  struct listener_impl : vendor_sdk::listener
  {
    vendor_session* owner;

    void on_vendor_execution(const vendor_sdk::exec_msg& msg) override
    {
      owner->on_execution_report(translate_execution(msg));
    }

    void on_vendor_reject(const vendor_sdk::reject_msg& msg) override
    {
      owner->on_rejected(translate_rejection(msg));
    }
  };

  vendor_sdk::client client_;
  listener_impl listener_{this};
};
```

The wrapper is the boundary at which the foreign vocabulary becomes the
domain vocabulary. `vendor_sdk::exec_msg` does not appear outside this
class; the rest of the program deals in `execution_report` and
`rejection`. The exact mechanism for subscribing to vendor callbacks
varies -- virtual interface, function-pointer registration, callback
struct -- the wrapper is the place where that mechanism becomes domain
callbacks.

`architecture.md`'s "Domain separation with adapters" covers the broader
rule: integrations live in dedicated adapter modules. The pipeline shape
is how an adapter exposes itself to the rest of the system -- through
`on_*` callbacks that speak the domain's language. Downstream stages do
not know whether the trigger came from a vendor SDK callback, a socket
read, or a test driver.

## Why this shape pays off

**Components compose without circular dependencies.** Stage A does not
know about stage B; stage B does not know about stage A. They exist as
producer-consumer relationships the wiring layer establishes. A callback
assignment in `main` is not a compile-time dependency from A to B or B
to A; see `architecture.md` "Forward-only dependencies".

**Test doubles substitute by capturing callbacks.** A unit test for the
strategy stage assigns `on_order_decision` to a lambda that pushes into
a `std::vector`, drives `send` with crafted market data, and asserts on
what came out. No mocking framework, no virtual interfaces. The probe
pattern in `../cpp-testing-principles/test-helpers.md` is the same idea
for internal state; for outbound callbacks, the assignment *is* the
substitution mechanism.

**The topology is visible in one place.** A new developer reads `main`
to see what the program does. The components describe themselves
through their inbound methods and outbound callbacks; the wiring
describes how they fit together. A change to the pipeline shape is a
change to one file.

**Effects sit at the edge.** Pipeline stages are pure transformations.
External wrappers and the wiring layer hold the I/O and the threading.
This is the realization of `architecture.md`'s functional core /
imperative shell at the data-flow level.

## Trade-offs

**Wiring grows quadratically if every component connects to every
other.** That is the failure mode of a global event bus where every
stage subscribes to every event type and filters at the receiver. The
pattern works because the producer-consumer pairs are domain-specific
and explicit. Resist generic event types like `event` or `message`
shared across stages: they make the wiring boilerplate disappear at the
cost of losing the type-checked, named-callback structure that
justifies the pattern.

**Synchronous unwind is a property of direct-call wiring.** A stage
emits, whose handler emits, all on one thread, all on one stack. For
short chains of pure stages this is exactly what the reader wants --
the topology in `main` is the call graph. When a stage can re-enter an
upstream stage, or when a stage holds a lock across emission, or when
stack depth becomes a concern, switch the wiring shape: post to the
consumer's loop, or queue locally (see "Interception at the wiring
point"). The unwind is the property of the lambda the wiring layer
chose, not of the pattern.

**Unassigned callbacks are a runtime hazard.** An `on_*` field the
wiring layer forgets to set will throw or crash the first time the
stage fires it. Two defenses: default-construct the callback to a
no-op where the absence of a subscriber is legitimate; require the
assignment in the wiring layer where it is not, and add a startup
assertion that every required callback was wired before the stage
runs. The choice is per-callback -- subscribers to optional side
events default to no-op; primary outputs are required and asserted.

**Type erasure has a cost.** Each `inplace_function` or `std::function`
call goes through an indirection the compiler cannot inline across. On
a measured hot path where the indirection matters, replace the
type-erased callback with a template parameter on the stage type, so
the consumer's call is a direct function call the compiler can inline.
This trades wiring flexibility for speed; reach for it only when a
profiler points at the indirection. See `performance.md`
"Type-erased callables on hot paths".

## See also

- `architecture.md` -- domain separation with adapters, forward-only
  dependencies, and the functional core / imperative shell that the
  wiring layer realises in practice.
- `runtime.md` -- event loops, the `post` primitive, and the broader
  treatment of where threads sit in the architecture. The wiring
  lambda is where `post` calls land when the consumer runs on a
  different thread than the producer.
- `state-machines.md` -- the deferred-event pattern this page's
  re-entrance queue mirrors at the wiring level.
- `cross-cutting.md` -- variant-based service handles chosen at
  startup; the same edit-at-the-edge idea, one level up from per-edge
  data flow.
- `../cpp-testing-principles/test-helpers.md` -- the probe pattern for
  intercepting internals in tests; outbound callback substitution is
  the same idea applied to the public surface.
- `performance.md` -- the inplace function variant for type-erased
  callables on hot paths.

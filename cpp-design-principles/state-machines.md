# State machines

A state machine turns implicit stateful logic into an explicit graph: a fixed
set of states, a fixed set of events, and a named transition for every legal
move. Where ad-hoc booleans and `if (state == X && pending_ && !closing_)`
branches scatter the rules across many call sites, an FSM concentrates them
in one place that can be read top-to-bottom, diffed, drawn, and tested edge
by edge.

This page describes when that trade is worth it, how to implement an FSM in
modern C++ with Boost.SML, how to keep an embedded PlantUML diagram aligned
with the transition table, and how to test the machine in isolation from the
class that owns it.

## When a state machine is the right tool

Reach for an FSM when behavior is **driven by a graph of legal moves** and most
of the following are true:

- **Behavior depends on a sequence, not just a snapshot.** The legal next
  action depends on what happened before. "After `connect` and before
  `established`, `send` must queue rather than transmit." A function looking
  only at the current arguments cannot decide correctly without consulting a
  mode flag, and that flag has to be set and cleared from elsewhere.
- **Side effects fire on transitions.** Entering a state schedules a timer,
  leaving it cancels the timer, an event in flight needs an acknowledgement.
  Entry and exit actions are first-class concepts in an FSM.
- **Illegal transitions need to be impossible, not "we'll remember not to do
  that".** A protocol spec, a connection lifecycle, a workflow with approval
  gates -- anywhere the cost of an invalid transition is real, the machine
  refusing the event up front beats a runtime assertion buried in a method.
- **Multiple correlated booleans are creeping in.** Flags like `connected_`,
  `handshaking_`, `closing_`, and `pending_` that must be set and cleared in
  lockstep across several methods encode a state machine by hand, with every
  illegal combination silently representable. See the discussion in `compile-time-correctness.md`
  and `invariants.md`; when that pattern appears, an FSM eliminates the
  invalid combinations by construction.
- **The shape needs to be communicated.** A reviewer, a new team member, or
  the author six months later should be able to see the lifecycle without
  reading every method. A diagram on the page is a far better answer than a
  prose description.

## When not to use one

State machines have a cost: a new vocabulary, a transition table to read, a
diagram to keep in sync, and a compile-time hit from the template-heavy library.
The cost is wrong for:

- **Single-bit state.** `running_` / `stopped_`, `cached_` / `not_cached_`.
  An FSM around one boolean is ceremony with no payoff.
- **Linear pipelines.** "Parse, then validate, then write" runs once per
  input and has no branches that loop back. A function that calls three
  steps in order is the right tool; an FSM only obscures it.
- **Pure data transformations.** Mapping, filtering, folding -- no state to
  speak of beyond the data being threaded through. See `declarative-style.md`.
- **Behavior fully captured by the type system.** When transitions are trivial
  and there are no entry/exit actions, `std::variant` can encode state without
  an FSM. See `compile-time-correctness.md`.
- **Behavior that does not yet need to be enumerated.** If the rules are
  still being discovered, a hand-written switch evolves faster than an FSM
  refactor. Reach for the FSM once the shape stabilizes.

A useful smoke test: try to draw the lifecycle on paper before writing
anything. If the drawing has fewer than three states and no entry/exit
actions, write it as a function or a `std::variant`. If it has loops,
retries, error recovery paths, or actions that fire on edges, an FSM will
pay for itself.

## Implementing with Boost.SML

[`boost::sml`](https://boost-ext.github.io/sml/) compiles a transition table
into a zero-allocation, branch-table dispatcher. Three things make it a good
default for production C++:

- **The transition table is the source of truth.** The DSL reads close
  enough to the diagram that a reviewer can check the two against each other
  by eye.
- **The FSM has no business logic.** Side effects are injected as a struct of
  callables, so the FSM only sequences them.
- **The `sml::testing` policy unlocks state injection** for tests, so each
  edge can be exercised in isolation without driving the machine through
  every preceding event.

Alternatives worth mentioning:

| Approach                       | Where it fits                                              |
|--------------------------------|------------------------------------------------------------|
| Hand-written `switch` on enum  | A few states, no entry/exit actions, low coordination cost |
| `std::variant` + `lib::match`  | State carries data, transitions are obvious from the types |
| Boost.MSM, Boost.Statechart    | Already in the codebase; otherwise compile times are heavy |
| `boost::sml`                   | Default choice for non-trivial FSMs in modern C++          |

### Anatomy of a transition table

```cpp
#include <boost/sml.hpp>

namespace gateway::detail::session_state_machine_fsm {

// Events: empty structs (or with payload fields when the event carries data).
struct ev_connect {};
struct ev_close {};
struct ev_established {};
struct ev_closed {};

// States: empty tag types.
struct st_closed {};
struct st_connecting {};
struct st_established {};
struct st_closing {};

// Actions: the only side effects the machine can produce. Each member is a
// type-erased callable that the owning class binds to a real implementation.
struct actions
{
  lib::inplace_function<void()> async_connect;
  lib::inplace_function<void()> async_close;
};

struct transitions
{
  auto operator()() const noexcept
  {
    namespace sml = boost::sml;
    namespace fsm = gateway::detail::session_state_machine_fsm;

    return sml::make_transition_table(
      *sml::state<st_closed> + sml::event<ev_connect> = sml::state<st_connecting>,

      sml::state<st_connecting> + sml::on_entry<sml::_>  / [](fsm::actions& a) { a.async_connect(); },
      sml::state<st_connecting> + sml::event<ev_established> = sml::state<st_established>,
      sml::state<st_connecting> + sml::event<ev_close> / [](fsm::actions& a) { a.async_close(); } = sml::state<st_closing>,

      sml::state<st_established> + sml::event<ev_close> / [](fsm::actions& a) { a.async_close(); } = sml::state<st_closing>,

      sml::state<st_closing> + sml::event<ev_closed> = sml::state<st_closed>
    );
  }
};

} // namespace gateway::detail::session_state_machine_fsm

namespace gateway::detail {

template <typename... Policies>
using session_state_machine =
    boost::sml::sm<session_state_machine_fsm::transitions, Policies...>;

} // namespace gateway::detail
```

DSL primitives used above:

| Pattern                            | Meaning                             |
|------------------------------------|-------------------------------------|
| `*sml::state<S>`                   | Initial state (leading `*`)         |
| `+ sml::event<E>`                  | Event trigger                       |
| `[guard]`                          | Guard predicate; must return `bool` |
| `/ action`                         | Action; must return `void`          |
| `= sml::state<D>`                  | Target state                        |
| `+ sml::on_entry<sml::_> / action` | Entry action                        |
| `+ sml::on_exit<sml::_>  / action` | Exit action                         |

Transition execution order during `S1 -> S2`: `on_exit(S1)` runs first, then
the transition action, then `on_entry(S2)`.

### Wiring the machine into its owner

The owner holds two members, in this order so initialization works:

```cpp
namespace fsm = gateway::detail::session_state_machine_fsm;

class session
{
public:
  void connect()         { fsm_.process_event(fsm::ev_connect{}); }
  void on_link_up()      { fsm_.process_event(fsm::ev_established{}); }
  void close()           { fsm_.process_event(fsm::ev_close{}); }
  void on_link_closed()  { fsm_.process_event(fsm::ev_closed{}); }

  [[nodiscard]] auto established() const -> bool
  {
    return fsm_.is(boost::sml::state<fsm::st_established>);
  }

private:
  // Actions struct first: binds FSM callbacks to private member functions.
  fsm::actions fsm_actions_{
    .async_connect = [this] { do_async_connect(); },
    .async_close = [this] { do_async_close(); },
  };

  // FSM next: holds a reference into fsm_actions_.
  detail::session_state_machine<> fsm_{fsm_actions_};

  void do_async_connect();
  void do_async_close();
};
```

Public methods on the owner translate external events into `process_event`
calls. Private `do_*` methods carry the side-effecting implementations the
FSM dispatches through its actions struct. Semantic accessors (`established()`)
expose state to callers.

## Deferred events

When an action calls `process_event` from inside a transition, the second
call is re-entrant: it begins a new transition while the first is still
mid-walk, with the same hazards as mutating a container while iterating
it. The fix is the one `pipelines.md` reaches for at the wiring layer in
"Defer re-entrant callbacks instead of recursing": queue the event now,
dispatch it after the current call stack unwinds.

Boost.SML provides two primitives that build on the same internal queue
mechanism:

- `sml::defer` is a transition action that puts the triggering event on
  a queue. SML re-processes the queue every time the state changes, so
  the deferred event retries against the new state. Use this when an
  event arrives in a state where it is not yet legal and the rule is
  "wait until we are somewhere it is".
- `sml::process(event{})`, called from inside an action, posts a
  follow-up event to be dispatched after the current `process_event`
  returns. Use this when a transition needs to emit a new event
  without re-entering the dispatcher on its own stack.

Both require opting in to a queue policy on the state machine type:

```cpp
namespace fsm = gateway::detail::session_state_machine_fsm;

template <typename... Policies>
using session_state_machine = boost::sml::sm<
    fsm::transitions,
    boost::sml::defer_queue<std::deque>,
    boost::sml::process_queue<std::queue>,
    Policies...>;
```

`defer_queue` and `process_queue` take a container template; SML
instantiates it once per event type that can be queued.

### Defer in the transition table

Extend the session FSM with a `send` request the application can issue
before the link is up. Without defer, the FSM either drops the event or
forces the caller to gate every send on `established()`. With defer,
the machine holds the event until the handshake completes.

```cpp
namespace sml = boost::sml;

return sml::make_transition_table(
  // ... existing rows ...
  sml::state<st_connecting> + sml::event<ev_send> / sml::defer,
  sml::state<st_established> + sml::event<ev_send> /
      [](fsm::actions& a) { a.write_message(); }
);
```

`ev_send` arrives while the session is mid-handshake; the `defer`
action enqueues it. When `ev_established` fires and the machine moves
to `st_established`, SML drains the queue and re-processes `ev_send`
against the new state. The second row matches, the write happens, and
the caller of `process_event(ev_send{})` never sees the wait.

### Process for follow-up events

`sml::process` solves the dual shape: an action needs to emit a new
event, but doing so synchronously would re-enter `process_event` from
inside itself.

```cpp
// BAD: re-enters process_event mid-transition.
sml::state<st_idle> + sml::event<ev_start> /
    [](fsm::actions& a, auto& sm) {
      a.preflight();
      sm.process_event(ev_ready{});
    } = sml::state<st_running>,

// GOOD: queued and dispatched after the current call returns.
sml::state<st_idle> + sml::event<ev_start> /
    [](fsm::actions& a, auto& sm) {
      a.preflight();
      sm.process(ev_ready{});
    } = sml::state<st_running>
```

`defer` waits for the next state change; `process` waits for the call
stack to unwind. Both keep the FSM's call graph acyclic.

### When a queue is worth it

A queue policy adds storage, an allocation per enqueue under the
default `std::deque`, and a re-dispatch step on every state change.
The cost is invisible on setup-and-teardown lifecycles -- sessions,
sagas, order flows firing a handful of events per second. It becomes
visible on hot paths: a market-data normalizer running at hundreds of
thousands of events per second pays the allocation on every defer.

Reach for a queue policy when:

- Events legitimately arrive in states where they are not yet valid,
  and the right behavior is to wait rather than drop or fail.
- An action needs to emit a follow-up event without re-entering
  `process_event` from its own stack.

Skip the queue policy and restructure the caller when:

- The FSM ticks faster than enqueue and dequeue overhead allows.
  Buffer at the caller's level, or split the FSM so the hot path
  does not need the queue.
- Events in the wrong state are a programming error. A missing
  transition surfaces the bug; a silent defer hides it.
- The "follow-up event" is really sequential logic. Two transitions
  back-to-back in the table read more clearly than a `process` call
  inside an action.

### Queue container choice

The container template parameter is the place to swap when the default
allocation cost shows up in a profile.

| Container         | When it fits                                                       |
|-------------------|--------------------------------------------------------------------|
| `std::deque`      | Default. Variable size, pointer stable, allocates per page.        |
| `plf::queue`      | Hot paths. Segmented storage, cache-friendly, faster push and pop. |
| Fixed ring buffer | Known bound on pending events; no allocation at runtime.           |

`plf::queue` (from the `plf::colony` family) presents the same
`push`/`pop`/`empty` surface as `std::queue` but lays its storage out
for tighter cache behavior. The win shows up when enqueue and dequeue
happen often enough to be visible in a profile; on a session FSM that
defers once per handshake, the difference does not register.

A fixed-capacity ring buffer is the answer when the maximum number of
in-flight events is known by construction -- for example, when the FSM
only ever defers one event between consecutive state changes. The
compile-time bound replaces the allocator and overflow becomes a
compile-time error.

See `performance.md` for the broader allocation-on-hot-path guidance;
the same reasoning that drives `lib::inplace_function` over
`std::function` drives the container choice here.

### Testing deferred transitions

`boost::sml::testing` composes with `defer_queue`; the state machine
type lists both policies.

```cpp
boost::sml::sm<
    fsm::transitions,
    boost::sml::testing,
    boost::sml::defer_queue<std::deque>>
    sm{ctx};
```

A defer test fires the event in its deferred state, drives the state
change that should release it, and asserts the released event's action
fired once and in the right order.

```cpp
TEST_CASE_METHOD(session_state_machine_fixture,
                 "session fsm - defers ev_send during connecting", "[session][fsm]")
{
  sm.process_event(fsm::ev_connect{});       // closed -> connecting
  sm.process_event(fsm::ev_send{});          // deferred; no write fires
  CHECK_ACTION(0, async_connect);
  REQUIRE(actions.size() == 1);

  sm.process_event(fsm::ev_established{});   // drains queue; ev_send retries
  CHECK_ACTION(1, write_message);
  REQUIRE(actions.size() == 2);
}
```

The deferred event needs no special test scaffolding; it shows up as
the absence of an action in step two and its appearance after the
state change in step three.

## Embedded PlantUML for documentation and verification

A transition table is precise but linear; a diagram makes the topology
visible at a glance. Embedding the diagram in the header, as a PlantUML
comment block immediately above the transition table, gives both audiences
what they need without splitting the source of truth across two files.

```cpp
/*
@startuml
skin rose
skinparam state {
  BackgroundColor #FEFEFE
  BorderColor     #555555
  ArrowColor      #555555
  FontSize        13
}

state Closed
state Connecting
state Established
state Closing

[*] -down-> Closed

Closed       -down->  Connecting   : connect
Connecting   :        on_entry / async_connect()
Connecting   -down->  Established  : established
Connecting   -right-> Closing      : close / async_close()
Established  -right-> Closing      : close / async_close()
Closing      -up->    Closed       : closed
@enduml
*/

struct transitions
{
  auto operator()() const noexcept { /* table mirrors the diagram above */ }
};
```

Conventions that keep the diagram useful:

- **Annotate entry and exit actions on the state line.** `Connecting :
  on_entry / async_connect()` reads as "while in this state, on entry, call
  `async_connect`". Transition actions go after the event label: `close /
  async_close()`.
- **Guards in square brackets.** `event [guard] / action`, matching the
  Boost.SML syntax.
- **The diagram and the transition table must match exactly.** Edges,
  guards, and actions all. The diagram is not a sketch; it is the
  specification the table implements. A divergence is a bug in one or the
  other.
- **Review them as a pair.** Every change to the transition table prompts a
  matching change to the PlantUML. Reviewers check both panels.

The diagram is human-readable, the table is machine-readable, and a
side-by-side reading is the cheapest correctness check available -- before
any test runs.

### Rendering the diagram

PlantUML source is plain text; rendering it is a separate step.

- **Inside the editor.** The VSCode extension `jebbs.plantuml` (and similar
  plugins in JetBrains, Vim, Emacs) renders fenced or comment-embedded
  PlantUML to a preview pane. Working from the header, a reviewer can see
  the diagram live without leaving the file.
- **Online.** The public PlantUML server (`plantuml.com/plantuml`) renders
  pasted source to PNG or SVG. Useful for quick sharing during review. Avoid
  it for diagrams whose state names disclose internal architecture you would
  not put on a pastebin.
- **In CI or docs builds.** `plantuml.jar` (or the `plantuml` Debian
  package) emits PNG/SVG from `.puml` files on the command line. A docs
  build can extract the comment block, pipe it through `plantuml`, and embed
  the rendered image in the generated site.

Pick whichever fits the workflow; the diagram is the artifact, not the
tooling around it.

## Testing a state machine

Once the FSM is isolated behind an actions struct, it tests cleanly without
any of the owning class's collaborators. The `sml::testing` policy adds
`set_current_states(state<S>)`, so each transition can be exercised from its
source state directly without driving the machine through every preceding
event.

The fixture records every action the machine fires into a
`std::vector<std::variant<...>>`, which lets assertions check both *which*
actions ran and *in what order*.

```cpp
#include <variant>
#include <vector>

#include <catch2/catch_test_macros.hpp>
#include "gateway/detail/session_state_machine.hpp"

namespace fsm = gateway::detail::session_state_machine_fsm;

namespace {

// One tracking struct per action in fsm::actions.
struct action_async_connect {};
struct action_async_close {};

using action = std::variant<action_async_connect, action_async_close>;

struct session_state_machine_fixture
{
  std::vector<action> actions;

  fsm::actions ctx{
    .async_connect = [this] { actions.emplace_back(action_async_connect{}); },
    .async_close = [this] { actions.emplace_back(action_async_close{}); },
  };

  gateway::detail::session_state_machine<boost::sml::testing> sm{ctx};

  template <typename State>
  [[nodiscard]] auto is_in_state() const -> bool
  {
    return sm.is(boost::sml::state<State>);
  }

  template <typename State>
  void set_state()
  {
    sm.set_current_states(boost::sml::state<State>);
    actions.clear(); // set_current_states fires entry actions; discard them so assertions see only test-driven events
  }
};

#define CHECK_ACTION(idx, name)                                                \
  REQUIRE(actions.size() > (idx));                                             \
  CHECK(std::holds_alternative<action_##name>(actions[idx]))

} // namespace
```

The tests then walk the diagram: every edge from every state, plus one full
lifecycle, plus one test per retry or error loop.

```cpp
TEST_CASE_METHOD(session_state_machine_fixture,
                 "session fsm - initial state is closed", "[session][fsm]")
{
  CHECK(is_in_state<fsm::st_closed>());
}

TEST_CASE_METHOD(session_state_machine_fixture,
                 "session fsm - connecting transitions", "[session][fsm]")
{
  sm.process_event(fsm::ev_connect{});
  CHECK(is_in_state<fsm::st_connecting>());
  CHECK_ACTION(0, async_connect);

  SECTION("established - moves to established")
  {
    sm.process_event(fsm::ev_established{});
    CHECK(is_in_state<fsm::st_established>());
  }

  SECTION("close - moves to closing and fires async_close")
  {
    sm.process_event(fsm::ev_close{});
    CHECK(is_in_state<fsm::st_closing>());
    CHECK_ACTION(1, async_close);
  }
}

TEST_CASE_METHOD(session_state_machine_fixture,
                 "session fsm - full lifecycle", "[session][fsm]")
{
  sm.process_event(fsm::ev_connect{});      // closed     -> connecting
  sm.process_event(fsm::ev_established{});  // connecting -> established
  sm.process_event(fsm::ev_close{});        // established -> closing
  sm.process_event(fsm::ev_closed{});       // closing    -> closed

  CHECK(is_in_state<fsm::st_closed>());
  CHECK_ACTION(0, async_connect);
  CHECK_ACTION(1, async_close);
  REQUIRE(actions.size() == 2);
}
```

The fixture pattern follows the factory and probe conventions in
`cpp-testing-principles/test-helpers.md`; the per-state `SECTION`
composition follows the patterns in
`cpp-testing-principles/test-patterns.md`. Two notes specific to FSM tests:

- **Test every edge in the diagram once.** Group by source state and use
  `SECTION` for the alternative events out of that state. If a state is
  unreachable from initial with a single event, jump to it with
  `set_state<>()` instead of repeating an event sequence in every test.
- **Test the owning class separately.** The FSM tests prove the transition
  table is correct in isolation. Integration tests on the owner prove that
  its events actually translate to the right `process_event` calls and that
  its `do_*` methods produce the right side effects. Mixing the two loses
  the diagnostic value of each.

## Anti-patterns

- **Business logic inside transition lambdas.** The lambda calls one method
  on the actions struct and returns. Anything more belongs in the owning
  class, where it can be tested and reasoned about without the FSM context.
- **Implicit transitions.** A `process_event` call that silently does
  nothing because no edge matches is a bug waiting for production. Either
  the edge is missing from the table, or the caller is dispatching an event
  that should not exist in this state.
- **Diagram-table drift.** A diagram that no longer matches the table is
  worse than no diagram: it actively misleads. Treat them as a single
  artifact in code review; if you change one, change the other in the same
  commit.
- **Nesting machines that could be flat.** Composite machines are powerful
  but obscure the topology. Flatten unless the substates genuinely share
  entry/exit semantics that would otherwise have to be duplicated.
- **Defer as a workaround for a missing transition.** `sml::defer` is for
  events that legitimately arrive in a state where they will become legal
  soon. An unhandled event silently swallowed by a defer rule, then
  reprocessed against a state that still does not handle it, masks the
  same bug the implicit-transition anti-pattern describes. Add the edge
  to the table or reject the event at the boundary instead.
- **Calling `process_event` from inside a transition action.** The second
  call re-enters the dispatcher mid-walk. Use `sml::process` for
  follow-up events; see "Deferred events" above.
- **Leaking `fsm_.is(state<...>)` to callers.** The state type names are
  implementation details; semantic accessors (`established()`, `closing()`)
  let the FSM refactor freely without rippling through call sites.

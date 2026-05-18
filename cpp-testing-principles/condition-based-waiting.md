# Condition-based waiting

Asynchrony, threading, and external processes make tests flaky when waits
are chosen by guess rather than driven by the condition they depend on.

**Core principle:** wait for the actual condition you care about, not a
guess about how long it takes.

## When to use this pattern

Use condition-based waiting whenever a test needs to wait for something
to become true:

- A worker thread to finish processing an item.
- A file to appear on disk.
- A subscription to receive a message.
- A subprocess to reach a ready state.
- An asynchronous callback to fire.

Apply this only when the asynchrony is intrinsic to the behavior under test.

## The core pattern

A `wait_for` helper polls a predicate until it returns true or a timeout
expires.

```cpp
template <typename Predicate>
void wait_for(Predicate condition,
              std::string_view description = "condition",
              std::chrono::milliseconds timeout = std::chrono::seconds{5})
{
  auto const start = std::chrono::steady_clock::now();
  while (!condition()) {
    if (std::chrono::steady_clock::now() - start > timeout) {
      throw std::runtime_error(
          std::format("timeout waiting for {} after {} ms",
                      description, timeout.count()));
    }
    std::this_thread::sleep_for(std::chrono::milliseconds{10});
  }
}
```

The timeout prevents the test from hanging forever if the condition never
becomes true -- a missing timeout is one of the most common ways for
"flaky" to become "the CI job is stuck".

Replacing an arbitrary sleep with a wait is a local transformation:

```cpp
// Fixed sleep.
std::this_thread::sleep_for(std::chrono::milliseconds{50});
auto value = subscriber.last_message();
REQUIRE(value.has_value());

// Predicate wait.
wait_for([&] { return subscriber.last_message().has_value(); },
         "subscriber received message");
auto value = subscriber.last_message();
REQUIRE(value.has_value());
```

The second version is faster when the operation completes quickly (no
forced 50 ms wait) and reliable when the operation completes slowly (it
polls until ready, up to the timeout). The description appears in the
timeout message, so failures name what was awaited.

## Common scenarios

| Goal                          | Predicate                                              |
|-------------------------------|--------------------------------------------------------|
| A flag becomes true           | `[&] { return ready.load(); }`                          |
| A count reaches a threshold   | `[&] { return queue.size() >= 5; }`                     |
| A compound condition          | `[&] { auto lock = std::lock_guard{m}; return obj.ready && obj.value > 10; }` |

For shared state, prefer `std::atomic` for simple flags or a
mutex-guarded read inside the predicate for complex state.

## Condition variables for producer/consumer

When the producing side can notify, `std::condition_variable` is more
efficient than polling -- the waiter wakes the moment the condition is
signaled rather than at the next poll interval.

```cpp
std::mutex m;
std::condition_variable cv;
bool ready = false;

// Producer
{
  std::lock_guard lock{m};
  ready = true;
}
cv.notify_one();

// Consumer (in the test)
{
  std::unique_lock lock{m};
  auto const signaled = cv.wait_for(
      lock, std::chrono::seconds{5}, [&] { return ready; });
  REQUIRE(signaled);
}
```

The predicate inside `wait_for` handles spurious wakeups: the wait does
not return until the predicate is true or the timeout expires.

A test fixture that needs multiple async events to complete can
encapsulate the wait so the test body stays readable:

```cpp
struct connection_fixture
{
  bool client_ready = false;
  bool server_ready = false;
  std::mutex m{};
  std::condition_variable cv{};

  void on_connected(bool is_client)
  {
    std::lock_guard lock{m};
    if (is_client) {
      client_ready = true;
    } else {
      server_ready = true;
    }
    cv.notify_all();
  }

  bool wait_for_both(std::chrono::milliseconds timeout)
  {
    std::unique_lock lock{m};
    return cv.wait_for(lock, timeout,
                       [this] { return client_ready && server_ready; });
  }
};

TEST_CASE("connection - handshake")
{
  connection_fixture fixture;
  // ... set up connections that call fixture.on_connected ...
  REQUIRE(fixture.wait_for_both(std::chrono::seconds{1}));
}
```

The mutex and predicate prevent missed wakeups: a waiter that starts after
the notification still observes the updated flags. Use `notify_all` when
more than one waiter may need to re-check the predicate.

## When an arbitrary sleep is actually correct

A fixed sleep is appropriate when the test is verifying timing behavior
-- the sleep itself is the assertion. The recipe combines a wait for the
triggering condition with a deliberate sleep for the timed behavior under test.

```cpp
component.start_timer();
wait_for([&] { return component.is_running(); }, "timer started");

// The timer fires every 100 ms; 250 ms covers two ticks plus margin.
std::this_thread::sleep_for(std::chrono::milliseconds{250});
CHECK(component.tick_count() >= 2);
```

Confirm the operation started, sleep for a duration derived from the documented timing, and comment the derivation.

## Common mistakes

**Polling too fast.** `sleep_for(1us)` between polls burns CPU and slows
down everything else running in the same process. 10 ms balances
responsiveness against CPU cost; go shorter only if the test verifies a
sub-10 ms behavior.

**No timeout.** A predicate that never becomes true hangs the test
forever. Locally this looks like the test is "still running"; in CI it
eventually hits the global timeout, and at that point the failure is
much harder to diagnose because there is no per-test error message.
Always include a timeout with a clear description.

**Caching stale state.** A predicate that captures a value before the
loop will see the same value on every iteration. Capture the *source*
of the value -- the object, the function -- and re-read inside the
predicate.

```cpp
// Wrong: captures the initial value once.
auto count = subscriber.message_count();
wait_for([&] { return count >= 5; });

// Right: re-reads the count each iteration.
wait_for([&] { return subscriber.message_count() >= 5; });
```

**Forgetting memory ordering.** A plain `bool` written by one thread and
read by another is undefined behavior. Use `std::atomic` or protect
access with a mutex.

## See also

- `qt-gui.md` -- Qt event-loop waits and GUI-specific async tests.
- `catch2-conventions.md` -- assertion choice and Catch2 registration.
- `../cpp-design-principles/runtime.md` -- the threading model these waits
  usually exercise.

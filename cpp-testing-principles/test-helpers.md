# Test helpers

Helpers that keep test bodies focused on the behavior under test by isolating construction, setup, and internal-state access from scenario-specific logic.

## Factory functions

Tests should contain only data that distinguishes one scenario from another.
Boilerplate object construction belongs in factories so the test body reads
like a behavioral statement, not a constructor reference.

Each factory family takes a parameter struct of `std::optional` fields merged left-to-right, so a caller states only the fields that matter for the scenario and every unset field falls back to a stable default.

```cpp
// Bare defaults.
auto default_request = make_http_request();

// Module preset (file-scoped constant).
auto post_request = make_http_request(json_post);

// Preset + selective override.
auto user_request = make_http_request(
    json_post,
    request_params{.path = "/users", .body = R"({"id":1})"});

// Derived objects inherit fields from a source object.
auto source_request = make_http_request(json_post);
auto retry = make_http_request(
    source_request,
    request_params{.headers = std::vector{header{"retry", "true"}}});
auto response = make_http_response(source_request);
auto error = make_http_response(
    source_request,
    response_params{.status_code = 500});
```

The parameter struct looks like this:

```cpp
struct request_params
{
  std::optional<std::string>           method{};
  std::optional<std::string>           scheme{};
  std::optional<std::string>           host{};
  std::optional<std::string>           path{};
  std::optional<std::vector<header>>   headers{};
  std::optional<std::string>           body{};
  std::optional<std::chrono::seconds>  timeout{};
};
```

File-scoped presets describe a scenario in one place:

```cpp
const auto json_post = request_params{
  .method = "POST",
  .scheme = "https",
  .host = "api.example.com",
  .path = "/v1/items",
  .headers = std::vector{header{"content-type", "application/json"}},
  .body = R"({})",
  .timeout = std::chrono::seconds{5},
};
```

The merge function applies overrides field by field, then folds the
parameter pack with a variadic fold expression:

```cpp
template <typename T>
concept optional_request_params =
    std::same_as<std::remove_cvref_t<T>, request_params>
    || std::same_as<std::remove_cvref_t<T>, http_request>;

inline request_params override_params(request_params result,
                                      const request_params& override)
{
  if (override.method)  result.method = override.method;
  if (override.scheme)  result.scheme = override.scheme;
  if (override.host)    result.host = override.host;
  if (override.path)    result.path = override.path;
  if (override.headers) result.headers = override.headers;
  if (override.body)    result.body = override.body;
  if (override.timeout) result.timeout = override.timeout;
  return result;
}

inline request_params override_params(request_params result,
                                      const http_request& source);

inline request_params merge_request_params(optional_request_params auto&&... args)
{
  request_params result{};
  ((result = override_params(result, std::forward<decltype(args)>(args))), ...);
  return result;
}
```

### When not to create a factory

A factory earns its place when it provides stable defaults for many fields,
composes presets, or derives related objects from a source. A factory that
only forwards its arguments to a constructor adds noise without value.

```cpp
// BAD - trivial wrapper that adds nothing over direct construction.
auto make_endpoint(std::string scheme, std::string host, int port)
{
  return endpoint{.scheme = std::move(scheme), .host = std::move(host), .port = port};
}
auto e = make_endpoint("https", "example.com", 443);
```

```cpp
// GOOD - designated initializers read as self-documenting construction; no factory needed.
auto e = endpoint{.scheme = "https", .host = "example.com", .port = 443};
```

### File-local wrappers

When several tests in a single file share a customization that the module
factories do not provide, define a wrapper next to them rather than repeating
it in every test:

```cpp
namespace {

auto make_test_request()
{
  request_params params{};
  params.host = k_test_host;
  params.timeout = k_test_timeout;
  return ::make_http_request(params);
}

}  // namespace
```

## Test data

Test data follows three rules:

1. **Deterministic.** No random values; tests must be reproducible. Factories
   already provide stable defaults; only override what the scenario requires.
2. **Descriptively named.** Prefer `k_expired_token` and `k_max_payload_size`
   over magic literals scattered across cases. Use `constexpr` file-level
   constants when the value appears in more than one test.
3. **Boundary-aware.** When testing a threshold, include at least three
   points: below, at, and above the boundary.

Name a value when it is reused or when its meaning is not obvious at the call site; inline literals otherwise.

## Fixtures

Catch2's `TEST_CASE_METHOD` lets a struct or class act as the per-test
fixture, with its constructor and destructor running before and after the
body. Use it when several tests share the same nontrivial setup or teardown:

```cpp
class library_fixture
{
public:
  library_fixture()
  {
    library::settings settings{};
    settings.config_dir(std::getenv("LIBRARY_CONFIG_DIR"));
    library::init(settings);
  }

  ~library_fixture() { library::shutdown(); }
};

TEST_CASE_METHOD(library_fixture, "parser - decodes input", "[parser]")
{
  // Library is initialized for the body, shut down on the way out.
}
```

A fixture that owns expensive global state requires serial execution to avoid
races; see [catch2-conventions.md](catch2-conventions.md) for the runner
configuration.

## Test probes

Behavioral tests should normally drive a class through its public API and
assert against observable outcomes; reaching into private state couples the
test to the implementation and turns refactors into cascading test edits.

Some assertions are not about behavior but about hygiene: did the container that held the request actually release it. These checks belong in tests -- a silent leak is exactly the kind of bug a unit test should catch -- but the container is an implementation detail. Adding a public accessor bloats the API with something callers have no reason to use. A `test_probe<T>` specialization gives a test direct, friend-level access to those internals without exposing them in the production API. Reach for it when the test is about internal state that production code should not need to see, not as a shortcut around an API that is awkward to drive.

### How it works

The primary template is an empty struct that the production headers can name
in a friend declaration. Tests provide an explicit specialization in their
own translation unit, and the friend declaration grants that specialization
access to the private members. A deduction guide on the constructor lets
callers write `test_probe probe{obj}` without repeating the type.

```cpp
namespace testing {

template <typename T>
struct test_probe
{
};

template <typename T>
test_probe(T&) -> test_probe<T>;

}  // namespace testing
```

The class under test names the specialization as a friend. The declaration
is one line in the private section -- the empty primary template means the
friendship costs nothing until a test specializes it:

```cpp
namespace cache {

template <typename Key, typename Value>
class lru_cache
{
public:
  // ... public API ...

private:
  friend struct testing::test_probe<lru_cache>;

  struct node;
  std::unordered_map<Key, node> entries_;
  intrusive_list<node>           lru_list_;
};

}  // namespace cache
```

The test file defines the specialization. It holds a reference to the instance and exposes the private members the tests need:

```cpp
namespace testing {

template <>
struct test_probe<cache::lru_cache<key, value>>
{
  using impl_t = cache::lru_cache<key, value>;

  test_probe(impl_t& impl) : impl_{impl} {}

  auto& entries() { return impl_.entries_; }
  auto& lru_list() { return impl_.lru_list_; }

private:
  impl_t& impl_;
};

}  // namespace testing

// Usage in a test.
cache::lru_cache<key, value> cache{};
testing::test_probe probe{cache};  // deduction guide picks the specialization
auto& entries = probe.entries();
```

A probe can also forward private member functions, not just data. When a class has internal transitions that the public API does not surface -- a nested helper type with its own state machine, an internal step that the owning class normally drives -- the probe can expose those entry points so tests can drive the transitions directly.

A probe specialization can also be a template. For a class template, the
probe specializes on the same parameter and works for every instantiation
the tests need:

```cpp
template <typename T>
struct test_probe<seqlock<T>>
{
  test_probe(seqlock<T>& impl) : impl_{impl} {}

  auto seq() const { return impl_.seq_.load(); }

private:
  seqlock<T>& impl_;
};
```

When a class contains other tracked objects, create nested probes from the
parent's accessors. A small helper keeps the boilerplate at the call site
short:

```cpp
auto make_probes(service::request_router& router)
{
  testing::test_probe probe{router};
  testing::test_probe routes_probe{probe.routes()};
  testing::test_probe pending_probe{probe.pending()};
  return std::tuple{probe, routes_probe, pending_probe};
}

TEST_CASE("request_router - dispatches request", "[request_router]")
{
  service::request_router router{};
  auto [probe, _, pending_probe] = make_probes(router);

  REQUIRE(router.add_route(route_id{"orders"}, handler));
  REQUIRE(router.dispatch(make_http_request(), source_id{"test"}));
  CHECK(pending_probe.entries().size() == 1);
}
```

### Direct state injection

When a test targets a specific branch (a duplicate-id rejection, a
quota-exceeded path), do not drag the system through a long sequence of valid
actions just to reach the prerequisite state. Inject the state through the
probe and exercise only the path under test.

`decltype` on a probe accessor lets the test name the private nested types
without coupling to their fully-qualified names:

```cpp
auto make_session_entry(testing::test_probe<auth::session_manager>& probe,
                        const auth::session& session)
{
  using entry_t =
    std::remove_reference_t<decltype(probe.sessions())>::mapped_type;

  return entry_t{
    .state          = impl::session_state::make(session),
    .expiry_handler = [](session_id) {},
  };
}

auto make_token_entry(testing::test_probe<auth::session_manager>& probe,
                      session_id id);

auto capture_error_code(auto&& call) -> auth::error_code;
```

```cpp
// GOOD - inject state directly; the test body focuses on the branch under test.
TEST_CASE("session_manager - refresh - duplicate token", "[session_manager]")
{
  auth::session_manager manager{};
  REQUIRE(manager.configure(config));

  testing::test_probe probe{manager};

  auth::session session{ /* ... */ };
  probe.sessions().emplace(session.id, make_session_entry(probe, session));
  probe.tokens().emplace(
      token_id{"dup_token"}, make_token_entry(probe, session.id));

  auth::refresh_request refresh{.token_id{"dup_token"}, /* ... */};

  auto ec = capture_error_code([&] { return manager.refresh(refresh); });
  CHECK(ec == auth::error_code::duplicate_token);
}
```

```cpp
// BAD - 80+ lines of ceremony to reach the same precondition.
TEST_CASE("session_manager - refresh - duplicate token - overbuilt setup",
          "[session_manager]")
{
  auth::session_manager manager{};
  REQUIRE(manager.configure(config));

  // step 1: login + acknowledge to register the session
  REQUIRE(manager.login(credentials, callback));
  REQUIRE(manager.process(session_started{ /* ... */ }));

  // step 2-3: more request/response round-trips ...
  // step 4: issue an initial refresh to create the token ...

  // finally, the actual test
  auto ec = capture_error_code([&] { return manager.refresh(duplicate_refresh); });
  CHECK(ec == auth::error_code::duplicate_token);
}
```

**Direct injection** when the test targets a specific error branch and the
setup actions are not themselves under test. **Action sequences** when the
test verifies end-to-end behavior and the intermediate transitions *are*
part of the contract -- lifecycle tests, state-machine traversal tests,
anything where the path matters.

## Test providers

Some components depend on global providers (a configuration registry, a
clock, a feature-flag service). Tests configure these providers with test
data and restore the production state on the way out.

Use an RAII guard that installs the provider on construction and restores the
previous provider on destruction. A bare assignment leaks into the next test
when `REQUIRE` short-circuits or the test throws.

```cpp
namespace {

auto make_test_provider()
{
  config::test_provider provider;
  provider.set("retries.max", 3);
  return provider;
}

struct provider_guard
{
  provider_guard(config::test_provider provider)
      : previous_{std::exchange(config::provider, std::move(provider))}
  {
  }

  ~provider_guard() { config::provider = std::move(previous_); }

private:
  config::provider_type previous_;
};

}  // namespace

TEST_CASE("component - needs extra configuration", "[component]")
{
  auto provider = make_test_provider();
  provider.set("feature.experimental", true);
  provider_guard guard{std::move(provider)};

  // ... test body ...
}
```

When several tests share the same provider configuration, wrap the guard in a fixture and use `TEST_CASE_METHOD` so the constructor and destructor run around each test body.

## See also

- `catch2-conventions.md` -- runner configuration and serial execution.
- `error-path-testing.md` -- tests that assert rollback after a helper drives
  an error path.

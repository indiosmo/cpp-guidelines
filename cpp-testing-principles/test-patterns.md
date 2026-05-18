# Test patterns

Pick the right Catch2 construct for a scenario and lay out table-driven tests when the same logic must run against many rows.

## Choosing a test pattern

Use the simplest pattern that avoids duplication. The list is ordered from
simplest to most structured.

**Separate `TEST_CASE`.** Each scenario has its own setup, its own
assertions, or exercises unrelated code paths.

```cpp
TEST_CASE("parse_int - success",       "[parse_int]") { /* ... */ }
TEST_CASE("parse_int - invalid input", "[parse_int]") { /* ... */ }
```

**`SECTION`.** Scenarios share setup but branch into different assertions.

```cpp
TEST_CASE("validate_password - enforces length rules", "[validation]")
{
  password_policy policy{.min_length = 8, .max_length = 64};

  SECTION("within bounds - ok")
  {
    CHECK(validate_password("correct_horse", policy));
  }

  SECTION("too short - reject")
  {
    CHECK_FALSE(validate_password("short", policy));
  }
}
```

**`GENERATE`.** Same logic, different scalar inputs.

```cpp
TEST_CASE("is_whitespace - recognizes all whitespace characters", "[chars]")
{
  auto c = GENERATE(' ', '\t', '\n', '\r');
  CHECK(is_whitespace(c));
}
```

**`TEMPLATE_TEST_CASE`.** Same logic across different types.

```cpp
TEMPLATE_TEST_CASE("container - push_back", "[container]",
                   std::vector<int>, std::deque<int>)
{
  TestType c;
  c.push_back(42);
  CHECK(c.size() == 1);
}
```

`TEMPLATE_TEST_CASE` pays off only when type substitution alone is enough. As
soon as the bodies diverge on syntax -- different call sites, different field
accessors, different factory functions -- the test ends up needing
`if constexpr` dispatch helpers, and those helpers usually cost more lines
than just writing two `SECTION`s.

```cpp
// BAD - dispatch helpers cost more than the template saves
template <typename>
inline constexpr bool dependent_false_v = false;

template <typename Event>
auto make_event(const door& d)
{
  if constexpr (std::same_as<Event, lock_event>)       return make_lock(d);
  else if constexpr (std::same_as<Event, unlock_event>) return make_unlock(d);
  else static_assert(dependent_false_v<Event>);
}

TEMPLATE_TEST_CASE("door - records every event", "[door]", lock_event, unlock_event)
{
  door test_door{};
  event_recorder recorder{};

  for (int i = 0; i < 3; ++i) {
    dispatch(test_door, make_event<TestType>(test_door));
  }
  REQUIRE(recorded_count<TestType>(recorder) == 3);
}
```

These patterns compose. A `TEST_CASE` can use `SECTION` at the top with
`GENERATE` inside a section, or `GENERATE` at the top with `SECTION` branches
below:

```cpp
TEST_CASE("tcp_connection - data transfer transitions", "[tcp]")
{
  // GENERATE at top: cover both peer roles.
  auto role = GENERATE(peer::client, peer::server);
  auto conn = make_established_connection(role);

  // SECTION branches: different transfer scenarios share the role setup.
  SECTION("partial send - connection stays established")
  {
    conn.send(payload_of_size(512));
    CHECK(conn.state() == tcp_state::established);
  }

  SECTION("peer closes - connection moves to close_wait")
  {
    conn.on_fin();
    CHECK(conn.state() == tcp_state::close_wait);
  }
}
```

This runs four sub-cases: client/partial, client/close, server/partial,
server/close.

## Compile-time tests

When the contract is type-level, test it at compile time. `static_assert`
fits concept conformance, negative construction checks, type traits, and
exhaustive visitor guards that should fail during compilation rather than at
runtime.

```cpp
static_assert(serializable<order_placed>);
static_assert(!serializable<raw_buffer>);

static_assert(std::constructible_from<types::symbol, std::string_view>);
static_assert(!std::constructible_from<types::symbol, const char*>);
```

For variant visitors, pair `static_assert` with a dependent false helper so a
new alternative forces the visitor to name its behavior:

```cpp
template <typename>
inline constexpr bool dependent_false_v = false;

lib::match(event,
  [](const order_placed&) {},
  [](const order_canceled&) {},
  []<typename T>(const T&) {
    static_assert(dependent_false_v<T>, "unhandled event alternative");
  });
```

## Table-driven testing

When the same logic has to run against many rows of inputs and expected
outputs, drive the rows through a Catch2 generator. Two layouts work; the
choice is about readability, not capability.

**Inline `GENERATE(table<...>)`** -- a few columns, all of a kind:

```cpp
#include <catch2/generators/catch_generators_all.hpp>

TEST_CASE("parse_int - accepts well-formed decimal input", "[parse_int]")
{
  // clang-format off
  auto [label, input, expected] =
    GENERATE(table<std::string, std::string, int>({
    // label            input          expected
    { "zero",           "0",                  0},
    { "positive",       "42",                42},
    { "negative",       "-17",              -17},
    { "leading zeros",  "00123",            123},
  }));
  // clang-format on
  CAPTURE(label);

  CHECK(parse_int(input) == expected);
}
```

**Separate vectors** -- many columns, or when input and expected rows have different shapes:

```cpp
TEST_CASE("normalize_date - rolls overflowed components", "[date]")
{
  struct test_input
  {
    std::string label;
    std::string outcome;
    int         year;
    int         month;
    int         day;
  };

  struct test_expected
  {
    int year, month, day;
    int day_of_week, day_of_year, iso_week;
  };

  // clang-format off
  static const std::vector<test_input> cases = {
    //  label                outcome                   year   month   day
    { "in range",          "no rollover",             2024,      3,    1},
    { "day overflow",      "rolls into next month",   2024,      1,   32},
    { "month overflow",    "rolls into next year",    2024,     13,    1},
    { "leap day",          "stays in february",       2024,      2,   29},
  };

  static const std::vector<test_expected> expectations = {
    // year   month   day   dow   doy   iso_week
    { 2024,      3,    1,    5,    61,         9},
    { 2024,      2,    1,    4,    32,         5},
    { 2025,      1,    1,    3,     1,         1},
    { 2024,      2,   29,    4,    60,         9},
  };
  // clang-format on

  REQUIRE(cases.size() == expectations.size());

  const auto i = GENERATE(range(std::size_t{0}, cases.size()));
  CAPTURE(cases[i].label, cases[i].outcome);

  auto [label, outcome, year, month, day] = cases[i];
  auto [exp_year, exp_month, exp_day,
        exp_dow, exp_doy, exp_iso_week] = expectations[i];

  const auto result = normalize_date(year, month, day);
  CHECK(result.year       == exp_year);
  CHECK(result.month      == exp_month);
  CHECK(result.day        == exp_day);
  CHECK(result.day_of_week  == exp_dow);
  CHECK(result.day_of_year  == exp_doy);
  CHECK(result.iso_week   == exp_iso_week);
}
```

Include a `std::string` label as the first field, and add an `outcome` field when rows describe both a scenario name and an expected behavioral outcome. Call `CAPTURE(label)` (and `outcome` if present) immediately after the structured binding so a failing row identifies itself. Wrap the table in `// clang-format off` / `// clang-format on` and keep columns aligned. Declare vector-of-struct data `static const` so each sub-case does not reconstruct the table. Guard separate vectors with `REQUIRE(cases.size() == expectations.size())` before `GENERATE` to catch a row mismatch up front. Use decimal notation for floats (`250.0`, not `250`) to keep types unambiguous.

Choose inline tables for ~2-6 columns when every column is the same type or trivially related (e.g., input/expected pair). Choose separate vectors when the row would be unreadably long, when inputs and expectations are meaningfully distinct groups, or when struct field names add documentation that positional columns cannot.

## See also

- `catch2-conventions.md` -- naming, tagging, registration, and assertion
  conventions.
- `condition-based-waiting.md` -- async tests that wait for observable
  conditions instead of fixed durations.

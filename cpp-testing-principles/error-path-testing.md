# Error-path testing

Modern C++ error handling splits across two styles, and tests have to cover
both:

- **Result-like return types** -- `lib::result<T>`, `std::expected<T, E>`, or
  a project-specific alias. The function returns a value that holds either a
  `T` on success or an error on failure.
- **Exceptions** -- the function returns `T` directly and throws on failure.

Both styles share the same test shape: exercise the success path, exercise
the failure path, and assert that any partial state mutated along the way is
rolled back when the operation fails. The mechanics differ in how the
outcome is inspected.

## Testing `expected`-style return types

**Success path.** Assert that the result holds a value, then unwrap and
check it.

```cpp
auto result = parse_int("42", 10);
REQUIRE(result);             // fails fast if the expected holds an error
CHECK(*result == 42);
```

When the value itself is not interesting, the boolean check stands alone:

```cpp
REQUIRE(cache.insert(key, value));
```

**Failure path.** Assert that the result holds an error, then inspect it
through `error()`.

```cpp
auto result = parse_int("not a number", 10);
REQUIRE_FALSE(result);
CHECK(result.error() == parse_error::invalid_input);
```

A complete test for a fallible function typically covers both paths in one
`TEST_CASE`, with `SECTION`s for the success branches and a separate section
for the error:

```cpp
TEST_CASE("parse_int - decimal input", "[parse_int]")
{
  SECTION("positive value")
  {
    auto result = parse_int("125255", 10);
    REQUIRE(result);
    CHECK(*result == 125255);
  }

  SECTION("negative value")
  {
    auto result = parse_int("-17", 10);
    REQUIRE(result);
    CHECK(*result == -17);
  }

  SECTION("non-numeric input - error")
  {
    auto result = parse_int("abc", 10);
    REQUIRE_FALSE(result);
    CHECK(result.error() == parse_error::invalid_input);
  }
}
```

Other result types differ only in spelling; the obligation is the same.

If the codebase ships a result-specific helper, prefer it -- it keeps every
test site concise and makes the assertion failure immediately point to the
unexpected error value rather than to the unwrap site.

## Testing exception-throwing code

When the function under test throws on failure, use Catch2's `_THROWS`
family. Use `REQUIRE_THROWS` when any exception is sufficient,
`REQUIRE_THROWS_AS` when the type matters, `REQUIRE_THROWS_MATCHES` when both
the type and a message pattern must hold, and `REQUIRE_NOTHROW` or
`CHECK_NOTHROW` for the success path. `CHECK_THROWS` variants let the test
continue after the assertion, following the same `REQUIRE`-vs-`CHECK`
convention.

```cpp
TEST_CASE("checked_multiply throws on overflow")
{
  CHECK_NOTHROW(checked_multiply(100, 100));

  REQUIRE_THROWS_AS(
      checked_multiply(std::numeric_limits<int>::max(), 2),
      std::overflow_error);

  using Catch::Matchers::ContainsSubstring;
  REQUIRE_THROWS_MATCHES(
      checked_multiply(std::numeric_limits<int>::max(), 2),
      std::overflow_error,
      Catch::Matchers::Message(ContainsSubstring("overflow")));
}
```

`CHECK_THROWS` variants exist for the cases where the test should continue
after the assertion. As with `REQUIRE` vs. `CHECK`, use `REQUIRE_THROWS`
when later assertions would not be meaningful if the throw did not happen.

## Exception safety and scope-guard rollback

A function that mutates state and then runs a fallible operation has to
roll back on failure. The canonical idiom is a scope guard that undoes the
mutation, dismissed on success. See `../cpp-design-principles/error-handling.md`
for the error vocabulary and `../cpp-design-principles/invariants.md` for
scope-guard rollback.

```cpp
// stub -- see ../cpp-design-principles/invariants.md for the full pattern
auto guard = make_scope_exit([&] { entries_.erase(request.id); });
// ... fallible work ... guard.dismiss();
```

Each guarded mutation is a test target. Trigger the failing branch, then
inspect the post-call state to confirm the rollback fired. Use whatever
observation point the type already exposes -- a public accessor, a test-only
getter, or a `friend` -- and assert directly on it.

```cpp
// GOOD - inspect state directly to confirm the rollback fired.
TEST_CASE("registry - invalid entry rolls back the insert")
{
  registry r{};
  // ... seed the registry so the next call must fail validation ...

  auto result = r.register_entry(bad_entry);
  REQUIRE_FALSE(result);

  // The failing branch left no trace.
  CHECK(r.entries().find(bad_entry.id) == r.entries().end());
}
```

```cpp
// BAD - asserts cleanup indirectly via a follow-up "good" call.
auto bad = r.register_entry(bad_entry);
REQUIRE_FALSE(bad);

auto good = r.register_entry(good_entry);
REQUIRE(good);
// This tests good_entry's behavior, not bad_entry's cleanup.
```

For exception-throwing code the structure is identical -- wrap the failing
call in `REQUIRE_THROWS_AS` and assert the same post-call state. The
rollback obligation belongs to the function under test; the error-handling
style only changes how the test observes the failure.

## See also

- `philosophy.md` -- contracts, invariants, and expected behavior.
- `test-patterns.md` -- structuring success and failure branches with
  `SECTION`.
- `../cpp-design-principles/invariants.md` -- the production rollback pattern
  these tests verify.

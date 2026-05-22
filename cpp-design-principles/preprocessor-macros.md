# Preprocessor macros

Macros are a last resort. Reach for them only when the alternative is
repeated boilerplate the type system cannot remove. A template, a
constexpr function, or a small ordinary function should win whenever one
will fit.

The shapes that genuinely need the preprocessor cluster around a few
cases: a propagation macro that early-returns from its caller, a
declaration helper that pastes a unique temporary name on each
invocation, an interface that has to dispatch on whether trailing
arguments are present. Boost.PP covers each shape; the rules below are
about reaching for it without dragging in more than the use site needs.

## Narrow sub-headers, not umbrellas

Include the single Boost.PP sub-header for each utility used, at the use
site. `boost/preprocessor/cat.hpp` for `BOOST_PP_CAT`, a separate
sub-header for each other utility. Do not include
`boost/preprocessor.hpp` -- the umbrella pulls every Boost.PP header
transitively, which inflates compile time wherever the macro is used and
hides which utility the macro actually depends on.

```cpp
// GOOD - one sub-header per utility, named at the use site
#include "boost/preprocessor/cat.hpp"

#define LIB_CHECK(r, ...) \
  BOOST_PP_CAT(LIB_DETAIL_CHECK, __VA_OPT__(_WITH_CONTEXT))(r __VA_OPT__(,) __VA_ARGS__)
```

```cpp
// BAD - the umbrella for one utility
#include "boost/preprocessor.hpp"

#define LIB_CHECK(r, ...) \
  BOOST_PP_CAT(LIB_DETAIL_CHECK, __VA_OPT__(_WITH_CONTEXT))(r __VA_OPT__(,) __VA_ARGS__)
```

## No project-local Boost.PP rename header

A wrapper header that renames Boost.PP utilities to project-specific
names hides the dependency, multiplies vocabulary the next reader must
memorize, and tends to pull the umbrella for one utility. Use Boost.PP
utilities directly at the use site instead.

```cpp
// BAD - the rename header replaces a two-line BOOST_PP_CAT recipe with
//       five project-specific macros, and pulls the umbrella for them
// lib/preprocessor.hpp
#include "boost/preprocessor.hpp"

#define LIB_DETAIL_TOKEN_PASTE(x, y, z) x ## y ## z
#define LIB_DETAIL_TOKEN_PASTE3(x, y, z) LIB_DETAIL_TOKEN_PASTE(x, y, z)
#define LIB_DETAIL_TMP(prefix, id) LIB_DETAIL_TOKEN_PASTE3(prefix, _result_tmp_, id)
#define LIB_DETAIL_SELECT(name, suffix) LIB_DETAIL_SELECT_PASTE(name, suffix)
#define LIB_DETAIL_SELECT_PASTE(name, suffix) name ## suffix
```

## Indirect three-token paste for argument expansion

`BOOST_PP_CAT(BOOST_PP_CAT(a, b), c)` performs a three-token paste with
one layer of indirection: the inner call expands its argument (a
`__COUNTER__` value forwarded from the caller macro) before the outer
concatenation appends the trailing token. Use this to generate unique
temporary names in same-line macro invocations.

```cpp
#include "boost/preprocessor/cat.hpp"

#define LIB_DETAIL_CHECK_WITH_CONTEXT_BODY(r, id, ...)                                     \
  {                                                                                        \
    auto&& BOOST_PP_CAT(BOOST_PP_CAT(check, _result_tmp_), id) = (r);                      \
    if (!BOOST_PP_CAT(BOOST_PP_CAT(check, _result_tmp_), id)) {                            \
      auto BOOST_PP_CAT(BOOST_PP_CAT(error, _result_tmp_), id) =                           \
        boost::leaf::on_error(lib::error{__VA_ARGS__});                                    \
      return lib::make_error(BOOST_PP_CAT(BOOST_PP_CAT(check, _result_tmp_), id).error()); \
    }                                                                                      \
  }
```

A flat `BOOST_PP_CAT(prefix, id)` would paste `id` literally rather than
expanding it; two `LIB_CHECK` invocations on the same line would then
collide on the same temporary name. The double-nested paste lets a
forwarded `__COUNTER__` produce a fresh suffix for each use.

## Variadic dispatch via `BOOST_PP_CAT(name, __VA_OPT__(_SUFFIX))`

With no trailing arguments, `__VA_OPT__` expands to nothing and the
paste collapses to `name`. With trailing arguments, the paste reaches a
suffixed body. Use this to give a single macro a no-context body and a
with-context body without exposing a second macro name to the caller.

```cpp
#define LIB_CHECK(r, ...) \
  BOOST_PP_CAT(LIB_DETAIL_CHECK, __VA_OPT__(_WITH_CONTEXT))(r __VA_OPT__(,) __VA_ARGS__)
```

`LIB_CHECK(r)` reaches the no-context body; `LIB_CHECK(r, ctx...)`
reaches the with-context body, both through one macro name.

## Domain macros are not preprocessor utilities

A macro that abstracts a project concept is a domain abstraction, not a
preprocessor utility, even when it expands through the preprocessor. A
macro that defines a field with its default, glues a namespace to its
JSON round-trip, or registers an error category lives with the domain
type it shapes:

```cpp
LIB_DEFAULTED_FIELD(quantity, types::quantity, types::quantity{0});
LIB_AUTO_JSON_PFR_NAMESPACE(application);
LIB_DEFINE_ERROR_CATEGORY(domain);
```

These do not migrate to a preprocessor-utility header just because they
expand through the preprocessor. The preprocessor is the mechanism; the
domain concept is what the name refers to.

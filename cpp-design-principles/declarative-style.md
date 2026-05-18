# Declarative style

Prefer declarative code over imperative loops. Concretely, this means:

1. **Decompose.** Break complex functions into focused helpers. Separate data
   extraction from validation from business rules.
2. **Work on simpler types.** Pass helpers only what they actually use. If
   a helper reads a single field, take that field -- or project the range
   to it at the call site -- rather than threading the whole struct
   through. The helper stays generic, easy to test, and free of
   dependencies on fields it never touches.
3. **Stage variables upfront.** Compute derived data before the logic that
   uses it. This separates "what we know" from "what we check".
4. **Use named predicates.** Replace inline conditions with descriptive
   lambdas or functions.
5. **Use ranges and algorithms.** Prefer `std::ranges` and `std::views` over
   manual index loops.
6. **Compose lazily; materialize once.** Build the pipeline as a view, then
   either iterate it or convert to a container at the end. One traversal,
   no intermediate vectors.

## Decompose

Cut a function until each piece does one thing the reader can name. The
verbs in the call site become the table of contents.

**Mean and variance, separately.** Two named primitives plus a one-line
assembler beat one function that runs two stateful loops and ties the
result struct to the procedure that computes it.

```cpp
// BAD - one function, two loops
stats compute(const std::vector<double>& xs) {
  double sum = 0;
  for (auto x : xs) sum += x;
  double mean = sum / xs.size();

  double sq = 0;
  for (auto x : xs) sq += (x - mean) * (x - mean);
  return {mean, sq / xs.size()};
}
```

```cpp
// GOOD - two helpers, one assembler
double mean(std::span<const double> xs) {
  return std::ranges::fold_left(xs, 0.0, std::plus{}) / xs.size();
}

double variance(std::span<const double> xs, double mu) {
  auto sq = xs | std::views::transform(
      [mu](auto x) { return (x - mu) * (x - mu); });
  return std::ranges::fold_left(sq, 0.0, std::plus{}) / xs.size();
}

stats compute(std::span<const double> xs) {
  const auto mu = mean(xs);
  return {.mean = mu, .variance = variance(xs, mu)};
}
```

Each helper is independently testable and takes a `std::span` so callers
can pass any contiguous range. The assembler reads as the recipe it is.

`std::ranges::fold_left` is C++23. In C++20, `std::accumulate` over the
iterator pair is the equivalent (`std::accumulate(xs.begin(), xs.end(),
0.0)`).

**Widget render = paint + flush.** Layout decision, paint, and commit are
three different concerns; naming each makes the side-effect visible at
the call site.

```cpp
// BAD - layout + paint + flush in one
void render(const widget& w, canvas& c) {
  auto bounds = compute_rect(w);
  c.fill  (bounds, w.bg);
  c.stroke(bounds, w.border);
  if (!w.text.empty())
    c.draw_text(center_of(bounds), w.text);
  c.flush();
}
```

```cpp
// GOOD - pure paint, then flush
void paint(const widget& w, rect bounds, canvas& c) {
  c.fill  (bounds, w.bg);
  c.stroke(bounds, w.border);
  if (!w.text.empty())
    c.draw_text(center_of(bounds), w.text);
}

void render(const widget& w, canvas& c) {
  paint(w, compute_rect(w), c);
  c.flush();
}
```

`paint` is now testable against a recording canvas without having to mock
the flush; `render` is the one place that decides when to commit.

**Sean Parent's "slide".** A fiddly two-rotate move-a-subrange operation
gets a name, and every caller now reads in English (adapted from Sean
Parent, _C++ Seasoning_, GoingNative 2013).

```cpp
// BAD - two rotates in the middle of caller code
// move [f, l) to position p, in place
if (p < f) {
  std::rotate(p, f, l);
} else if (l < p) {
  std::rotate(f, l, p);
}
// ... caller continues, having open-coded "slide"
```

```cpp
// GOOD - named primitive
template <typename It>
auto slide(It f, It l, It p) -> std::pair<It, It> {
  if (p < f) return {p, std::rotate(p, f, l)};
  if (l < p) return {std::rotate(f, l, p), p};
  return {f, l};
}
```

Parent's rule of thumb: any composition you would reach for twice belongs
in a named function.

**Message handler: parse, validate, dispatch.** Three concerns share one
function; naming each stage lets a composer handle errors uniformly
instead of every clause logging on its own.

```cpp
// BAD - parse, validate, dispatch tangled in one function
void handle(const std::string& raw) {
  json j = json::parse(raw);
  if (!j.contains("type")) { log("missing type"); return; }
  if (j["type"] == "order") {
    if (!j.contains("qty") || j["qty"] <= 0) { log("bad qty"); return; }
    if (!j.contains("symbol"))               { log("missing symbol"); return; }
    send_order(j);
  } else if (j["type"] == "cancel") {
    if (!j.contains("order_id")) { log("missing order_id"); return; }
    send_cancel(j);
  }
}
```

```cpp
// GOOD - three named stages, one composer
lib::result<message> parse   (std::string_view raw);
lib::result<void>    validate(const message&);
void                  dispatch(const message&);

void handle(std::string_view raw) {
  leaf::try_handle_all(
    [&]() -> lib::result<void> {
      BOOST_LEAF_ASSIGN(auto msg, parse(raw));
      BOOST_LEAF_CHECK(validate(msg));
      dispatch(msg);
      return {};
    },
    [](const error::bad_qty& e) { log(e); },
    [] { log("unknown parse/validate error"); });
}
```

Each stage's signature says what it consumes and what it produces; the
composer is the only place that knows how errors flow. See
`error-handling.md` for the structured-error vocabulary.

## Work on simpler types

Pass a helper only what it actually uses. The narrower the input, the more
the helper is reusable, and the less of the surrounding world it has to
reason about.

**A ratio needs two numbers, not the resource that produced them.** The
calculation reads two counters; the struct that holds them owns a socket
and a logger that the calculation never touches. Taking the scalars
directly keeps the helper independent of the struct's other fields and
of their construction cost.

```cpp
// BAD - takes the whole session, even though only two counters matter
struct session {
  std::unique_ptr<tcp_connection> conn;
  std::unique_ptr<logger>         log;
  std::size_t bytes_sent;
  std::size_t bytes_acked;
};

double ack_ratio(const session& sess) {
  if (sess.bytes_sent == 0) return 0.0;
  return double(sess.bytes_acked) / double(sess.bytes_sent);
}
```

```cpp
// GOOD - takes only what it uses
double ack_ratio(std::size_t sent, std::size_t acked) {
  if (sent == 0) return 0.0;
  return double(acked) / double(sent);
}
```

The good version is callable with two literals; its unit test fits on one
line. Any pair of counters with the same meaning can use it, not just the
ones that happen to live on `session`.

**Find min by a field, not by points.** A projection lets the algorithm
name what it is doing rather than forcing the reader to parse a comparator
lambda (adapted from Barry Revzin, _Projections are Function Adaptors_,
brevzin.github.io, 2022).

```cpp
// BAD - lambda compares two points
point p = std::ranges::min(points,
  [](point const& a, point const& b) {
    return a.y < b.y;
  });
```

```cpp
// GOOD - projection reads as "by y"
point p = std::ranges::min(
    points, std::ranges::less{}, &point::y);
```

## Stage variables upfront

Compute the derived data first; run the logic that uses it second. The
function reads as "here is what we know, here is what we do with it"
rather than weaving derivation through control flow.

**Pagination -- derive once, assemble once.** Several fields share one
derived value (`pages`); compute it on its own line and the struct
initializer reads like a table.

```cpp
// BAD - derive inside the assembly
response resp;
resp.total = total;
resp.page  = query.page;
resp.size  = query.page_size;
resp.pages = (total + query.page_size - 1) / query.page_size;
resp.has_next = query.page < resp.pages;
resp.has_prev = query.page > 1;
return resp;
```

```cpp
// GOOD - derive first, build last
const int pages = (total + query.page_size - 1) / query.page_size;

return {
  .total    = total,
  .page     = query.page,
  .size     = query.page_size,
  .pages    = pages,
  .has_next = query.page < pages,
  .has_prev = query.page > 1,
};
```

The bad version sneaks `resp.pages` between its definition and its uses,
so the reader has to re-check that `resp.has_next` sees the value just
written. Naming `pages` once removes that step.

**Pipeline as named stages.** A simulation update written as a sequence
of named stages; each line's output is the next line's input, so the
function reads top to bottom as a recipe (adapted from Nikita Cherniy,
_Functional programming in C++ by example_, nikitablack.github.io).

```cpp
// BAD - mutate shapes in place across calls
shapes = updatePositions(shapes);
shapes = updateBounds(shapes);
shapes = updateCells(shapes);
```

```cpp
// GOOD - one named stage per step
auto s1 = calculatePositionsAndBounds(delta_time, shapes);
auto [rows, cols] = getNumberOfCells(width, height);
auto s2 = calculateCellsRanges(width, height, rows, cols, s1);
auto s3 = fillGrid(width, height, rows, cols, s2);
auto velocities = solveCollisions(s3, cols);
return applyVelocities(s3, velocities);
```

The bad version reuses one name for several distinct values; the reader
has to track what `shapes` means at each line. The good version gives
each intermediate its own identifier.

## Named predicates

Lift inline conditions into named lambdas or functions. The name carries
the intent; the body just spells it out, and the call site reads as a
sentence.

**is_active_user.** "Not deleted, and seen in the last 30 days" is a
domain concept that deserves a name rather than an inline composite
condition.

```cpp
// BAD - inline composite condition
auto active = std::ranges::count_if(users,
    [now](const auto& u) {
      return !u.deleted_at.has_value()
          && u.last_login > now - std::chrono::days(30);
    });
```

```cpp
// GOOD - the predicate has a name
const auto is_active = [now](const auto& u) {
  return !u.deleted_at.has_value()
      && u.last_login > now - std::chrono::days(30);
};

auto active = std::ranges::count_if(users, is_active);
```

**by_dept_then_seniority.** Any lambda more complex than a simple
`f(g(x))` composition deserves a name; below that threshold an anonymous
lambda is fine (adapted from Sean Parent, _C++ Seasoning_, GoingNative
2013).

```cpp
// BAD - multi-clause anonymous lambda
std::ranges::sort(employees, [](const auto& a, const auto& b) {
  if (a.department != b.department) return a.department < b.department;
  if (a.seniority  != b.seniority ) return a.seniority  > b.seniority;
  return a.last_name < b.last_name;
});
```

```cpp
// GOOD - the concept has a name
constexpr auto by_dept_then_seniority = [](const auto& a, const auto& b) {
  if (a.department != b.department) return a.department < b.department;
  if (a.seniority  != b.seniority ) return a.seniority  > b.seniority;
  return a.last_name < b.last_name;
};

std::ranges::sort(employees, by_dept_then_seniority);
```

**Predicate factory for a captured threshold.** A function that returns a
predicate beats writing the same shape with a different constant at every
call site; each call now reads as "greater than this threshold" with no
body to parse (adapted from Jonathan Boccara, _Out-of-line Lambdas_,
fluentcpp.com, 2020).

```cpp
// BAD - repeat the body every time
auto big_a = std::ranges::count_if(xs, [](int x) { return x > 100; });
auto big_b = std::ranges::count_if(ys, [](int x) { return x > 100; });
auto big_c = std::ranges::count_if(zs, [](int x) { return x > 200; });
```

```cpp
// GOOD - factory captures the threshold
auto greater_than(int threshold) {
  return [threshold](int x) { return x > threshold; };
}

auto big_a = std::ranges::count_if(xs, greater_than(100));
auto big_b = std::ranges::count_if(ys, greater_than(100));
auto big_c = std::ranges::count_if(zs, greater_than(200));
```

## Use ranges and algorithms

If the standard library already ships an algorithm that names what you
mean to do, reach for it instead of writing the loop.

**Any expired order?** A flag-and-`break` loop becomes one `any_of` call
with a named predicate; the algorithm names the primitive ("is there an
element satisfying P?"), the predicate names the question.

```cpp
// BAD - flag and break
bool has_expired = false;
for (const auto& o : orders) {
  if (o.expiry < now) {
    has_expired = true;
    break;
  }
}
if (has_expired) {
  return reject::expired_order_present;
}
```

```cpp
// GOOD - any_of with a named predicate
const auto is_expired = [now](const auto& o) { return o.expiry < now; };

if (std::ranges::any_of(orders, is_expired)) {
  return reject::expired_order_present;
}
```

**First missing required field.** `find_if` returns the iterator;
formatting the message is one step later. Two concerns -- searching and
formatting -- run in sequence rather than sharing a loop body.

```cpp
// BAD - index loop, error formatting tangled with the search
for (std::size_t i = 0; i < fields.size(); ++i) {
  if (fields[i].required && !fields[i].value) {
    return std::format("field {} is missing", fields[i].name);
  }
}
return std::nullopt;
```

```cpp
// GOOD - find_if returns the iterator, error formatting is one step later
constexpr auto is_missing = [](const auto& f) {
  return f.required && !f.value;
};

const auto it = std::ranges::find_if(fields, is_missing);
if (it == fields.end()) return std::nullopt;
return std::format("field {} is missing", it->name);
```

**Replacing a hand-rolled search.** When the loop is a search,
`std::ranges::find_if` names the primitive directly.

```cpp
// BAD - hand-rolled loop searching for the first active session
session* found = nullptr;
for (auto& sess : sessions) {
  if (sess.is_active()) {
    found = &sess;
    break;
  }
}
```

```cpp
// GOOD - algorithm names the operation
auto it = std::ranges::find_if(sessions,
    [](const auto& sess) { return sess.is_active(); });
```

## Compose lazily; materialize once

A pipeline of M operations applied eagerly traverses the data M times and
allocates M-1 intermediate containers. The same pipeline composed as views
produces each final element on demand: one logical traversal, no
intermediates. Reach for materialization (a `std::vector`, a
`std::ranges::to` conversion) only at the boundary where the result has to
outlive the pipeline or be handed to a non-range API.

The "stage variables upfront" principle applies at the pipeline level too:
compose the full view expression first, then iterate or materialize once at
the end.

**Eager pipeline pays M times.** Each stage allocates a vector and walks the
input again; an early `take` does nothing to save the cost of the stages
before it.

```cpp
// BAD - three allocations, three full passes; take(10) runs after
//       transform has already produced N strings.
std::vector<order> filtered;
std::copy_if(orders.begin(), orders.end(),
             std::back_inserter(filtered), is_buy);

std::vector<std::string> formatted;
std::transform(filtered.begin(), filtered.end(),
               std::back_inserter(formatted), to_csv_row);

std::vector<std::string> first10(formatted.begin(),
                                 formatted.begin() + std::min<std::size_t>(10, formatted.size()));
```

```cpp
// GOOD - one composed view; only the first 10 surviving rows are
//        ever formatted, and no intermediate vector exists.
auto rows = orders
          | std::views::filter(is_buy)
          | std::views::transform(to_csv_row)
          | std::views::take(10);

for (const auto& row : rows) write(row);
```

The view runs `filter`, then `transform`, then `take` element by element.
`take(10)` short-circuits the iteration as soon as ten elements reach it,
so `transform` never runs on the eleventh buy and beyond -- and the input
beyond that point is never even visited.

**Materialize when the result has to outlive the view.** A view borrows
from its input; once the input goes out of scope, iterating the view is
undefined behavior. When the result needs to be returned, stored, or
handed to an API that wants a container, convert with `std::ranges::to` at
the seam.

```cpp
// GOOD - C++23: one expression composes and materializes
auto active_ids = users
                | std::views::filter(is_active)
                | std::views::transform(&user::id)
                | std::ranges::to<std::vector>();
```

`std::ranges::to` is C++23. In C++20 the equivalent is constructing the
container from the view's iterators (or `begin/end`):

```cpp
// C++20 equivalent
auto active_view = users
                 | std::views::filter(is_active)
                 | std::views::transform(&user::id);
std::vector<user_id> active_ids(active_view.begin(), active_view.end());
```

Either form keeps the pipeline declarative and pushes the one allocation
out to the call site where the materialized value is actually used.

**Cost model, briefly.** For a pipeline of M stages over N elements:

| Form | Traversals | Allocations | Short-circuits? |
|------|------------|-------------|-----------------|
| Eager (M vectors) | O(N*M) | M-1 vectors | No -- every stage runs to completion before the next starts. |
| Lazy view | O(N) | 0 | Yes -- `take`, `find_if`, `any_of` stop pulling as soon as the answer is known. |
| Lazy + `ranges::to` at end | O(N) | 1 (the final container) | Yes, up to the materialization point. |

The performance argument and the readability argument point the same way:
build the description of what you want, then run it once.

See "Functional core, imperative shell" in `architecture.md` for where
effects live relative to these declarative inner layers.

Declarative composition is the default. See `performance.md` for the cases
where a measured hot path with a known budget warrants departing from it.

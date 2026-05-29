# Comments

Comments are part of the design surface. Names, types, and helper functions
should carry most of the model, but they do not always make the reader's next
step cheap. A good comment gives the reader the intent before they parse the
C++ syntax. It also gives reviewers a statement to check against the
implementation: if the code does not do what the comment says, one of them is
wrong.

Use comments to guide a reader through non-obvious lines and blocks. The goal
is not to narrate every statement. The goal is to keep intent close to the code
where the reader needs it.

## Guiding comments

A guiding comment is a short English statement of intent placed just before the
line or block it describes. It should let the reader skim a function as a
procedural walkthrough before reading the details.

```cpp
// reject duplicates and resolve the destination book up front.
BOOST_LEAF_CHECK(check_duplicate(incoming_key));
BOOST_LEAF_ASSIGN(auto* book_ptr, find_book(request.instrument));
auto& book = *book_ptr;

// ack precedes trades so receivers see the order id before fills on it.
on_event(order_ack{
  .user = types::user_id{request.user},
  .order_id = types::user_order_id{request.order_id},
});
```

The first comment names the step. The second comment also explains the ordering
constraint. Both help a reviewer check whether the implementation matches the
intended protocol.

## Non-obvious lines

Comment a single line when its intent is not obvious from the surrounding
names. Common causes are dense punctuation, designated initializers across
domains, chained calls, iterator invalidation rules, lifetime contracts,
performance-sensitive ordering, or a domain rule that the line relies on.

```cpp
// fully-filled maker: drop its identity index entry and return the slot to the pool.
resting_index_.erase(types::order_key{.user = consumed->data.user, .order_id = consumed->data.order_id});
release_node(consumed);
```

The comment is useful because the line is doing more than "erase": it ties a
domain event (a maker fully filled) to an ownership action (returning a pooled
node).

Iterator invalidation rules are another common source of non-obvious lines. When
correctness depends on a container's invalidation guarantee, state the guarantee
so a reviewer can verify the pattern against the standard.

```cpp
// safe to erase by value: unordered_set::erase invalidates only iterators to
// the erased element; orderIdIt already points past it ([unord.req.general]/14).
std::string orderId = *orderIdIt;
++orderIdIt;
// ...
book.removeOrder(orderId, order);
```

Without the comment, a reader must recall whether `std::unordered_set::erase`
invalidates all iterators (as rehash would) or only the erased one. The standard
guarantee is load-bearing here: if the container were a `std::vector`, the same
loop would be undefined behavior.

Do not comment one-line idioms that are already part of the local language.
For example, `lib::match` or `mil::match` dispatch often looks syntactically
busy, but in these codebases it is a repeatable convention. A simple dispatcher
does not need a comment.

```cpp
// BAD - repeats the local variant-dispatch idiom.
// dispatch to the typed handle overload.
mil::match(cmd, [this](const auto& cmd) { handle(cmd); });

// GOOD - the idiom carries itself.
mil::match(cmd, [this](const auto& cmd) { handle(cmd); });
```

The exception is when the `match` body itself encodes a non-obvious choice: a
fallible conversion, a protocol rule, a lifetime constraint, or a deliberately
partial subscription. Comment that choice, not the fact that variant dispatch
is happening.

## Blocks and flow

Lean toward commenting blocks. A block can be a group of statements, a loop, a
conditional branch, a lambda body, or a whole phase inside a function. Even
when the syntax is simple, reading one sentence is faster than reading the
loop. Consecutive block comments should read as the function's table of
contents.

```cpp
// rest the residual: pool-allocate a node, link it into the book,
// register its identity for future cancels and duplicate checks.
order_node* node = allocate_node(order);
book.place(node);
resting_index_.emplace(incoming_key, node);
```

```cpp
// walk the side map from best outward; consume each crossing level until
// the taker quantity runs out or the next level no longer crosses.
auto it = side_map.begin();

while (it != side_map.end() && taker.remaining_quantity > 0 && crosses(taker, it->first)) {
  const auto level_trades = consume_level(it->second, it->first, taker, on_trade, on_release);
  // ...
}
```

Block comments are especially valuable in long functions that are already
structured as phases. If the comment names a phase but the block contains
several unrelated tasks, split the block or extract a helper.

Use blank lines where they make the phases of a function easier to scan. They
are usually helpful between setup and a loop, after a multi-line control block
before the next independent statement, between non-trivial `switch` case groups,
and before a final `return` that follows a loop or branch. Keep tightly coupled
statements adjacent: assign-and-test pairs, a lookup and the `if` that checks
it, and guard bodies with a log and early `return`. Small helpers that set or
mutate one value and return it can stay compact.

## What not to comment

Do not narrate obvious code.

```cpp
// BAD - says only what the token already says.
// increment the counter.
++counter;
```

Do not keep a trail of past decisions in comments. Comments describe the
present code and the reason for its present shape. They do not preserve
rejected alternatives, migration notes, what the code used to do, what it no
longer does, or what another layer now handles. Git history is the record of
old designs; an ADR is the home for enduring decision rationale.

```cpp
// BAD - records a migration, not present intent.
// dedup moved to the upstream stage.
process(order);
```

If a precondition is real and load-bearing, phrase it as a present-tense
contract the code relies on.

```cpp
// BAD - frames a chore for the caller.
// caller must hold the session mutex.
auto next = session_.next_seq++;

// GOOD - states the invariant that makes this line safe.
// precondition: session mutex is held; next_seq is read and incremented without locking here.
auto next = session_.next_seq++;
```

Likewise, do not list non-responsibilities. A converter comment should say what
it converts, not enumerate routing, storage, logging, and status resolution that
happen somewhere else.

Do not paraphrase how standard library or framework internals work. The comment
should state the block's intent in the function's own terms, not narrate the
implementation of the tools it uses.

```cpp
// BAD - describes how views::split is implemented.
// std::views::split on a contiguous_range yields contiguous subranges;
// each one materialises into a string_view without copying input bytes.
for (const auto subrange : raw | std::views::split(separator)) {

// GOOD - names what the loop does in the function's terms.
// iterate each token, look it up in the table, and set the corresponding bit.
for (const auto subrange : raw | std::views::split(separator)) {
```

Do not reassert behavior that a wrapper, macro, or idiom already carries. A
line wrapped in `BOOST_LEAF_ASSIGN` is visibly failable, and a `result<T>`
return already signals propagation. Comment the step, not the error-handling
mechanics.

```cpp
// BAD - restates the failable nature and the propagation policy.
// Linear scan by token projection. mil::find_element raises
// mil::error_code::not_found, which we let propagate without dressing up.
BOOST_LEAF_ASSIGN(const auto& entry, mil::find_element(table, token, &row_type::token));

// GOOD - names the step.
// map token to flag.
BOOST_LEAF_ASSIGN(const auto& entry, mil::find_element(table, token, &row_type::token));
```

Prefer terse phrasing. When a short phrase carries the same intent as an
exhaustive enumeration, use the short phrase. The reader can recover the cases
from the code; they cannot recover the intent.

```cpp
// BAD - enumerates the cases the guard catches.
// skip leading, trailing, and doubled separators.
if (segment.empty()) {
  continue;
}

// GOOD - names what the guard is for.
// skip whitespace.
if (segment.empty()) {
  continue;
}
```

## Shape

Prefer `//` for guiding comments inside function bodies. Use `/* ... */` for
class, struct, file, and larger API documentation. Keep comments close to the
code they describe, and keep them short enough that the code still remains the
main artifact.

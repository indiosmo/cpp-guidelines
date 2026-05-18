# Conformance tests with ApprovalTests

Some outputs are long, structured blocks of text that are impractical to
encode as inline assertions: pretty-printer output, code-generator output,
JSON or XML serialization, report renderings, log formatting, AST dumps,
wire-level protocol messages. For these, ApprovalTests.cpp is a good fit.
The test renders the output, the framework diffs it against an approved
file checked into the repository, and the test fails if they disagree.

The approved file is the test's expected value: a reviewable artifact that
travels with the code and shows, in human-readable form, what the system
produces.

## When ApprovalTests fits

Approval tests fit when:

- **Large, structured output.** Twenty fields per record, ten records per
  scenario. An equivalent field-by-field assertion would bury the intent
  of the test under boilerplate, and a missing field would be easy to
  overlook in code review.
- **Evolving shape.** New fields, new sections, and reordered groups are
  normal during development. An approved file regenerates in one command;
  a hand-written expected value needs surgery every time.
- **Human oracle.** The expected output is "what the spec says this
  should look like" or "what a reviewer would recognize as correct"
  rather than a value computed from a formula. The reviewer reads the
  approved file and signs off; the test then locks it in.
- **Many parallel scenarios.** Encoding twenty FIX messages or thirty
  JSON payloads inline drowns the test in literals. One approved file
  with one section per scenario reads as a catalog.
- **Documentation by example.** The approved file doubles as a living
  sample of what the code emits. Readers learning the module open the
  approved file before the source.
- **Legacy characterization.** Capture the current behavior as an
  approved file, refactor, and let the diff tell you what changed.

## When to reach for something else

- **Single scalar or short string.** A direct `CHECK(parse(input) == 42)`
  is clearer than a one-line approval file, and the failure message names
  the actual mismatch instead of pointing at a diff.
- **Narrow assertion.** "Cancelling an unknown order returns
  `error_code::not_found`" is a one-line assertion. Wrapping it in a
  rendered string and diffing against an approved file obscures what the
  test is actually checking.
- **Non-deterministic output.** If timestamps, generated identifiers,
  iteration order, or thread interleaving leak into the output and you
  cannot reliably scrub or sort them, the test will fail spuriously and
  erode trust. Either tame the determinism first or write a different kind
  of test.
- **High churn format.** When every small change produces a sprawling
  diff, reviewers stop reading and start rubber-stamping. An approved file
  that nobody actually reads is worse than no test at all, because it
  lends false confidence. Split the rendering, narrow the test, or replace
  it with focused assertions.
- **Behavior, not output.** State-machine transitions, side effects on a
  collaborator, ordering of callbacks. Drive the system and assert against
  observable outcomes; do not stringify the state and diff it.
- **Performance-sensitive path.** Rendering strings and reading approved
  files costs more than a direct assertion. For tight loops in property or
  fuzz-style tests, prefer focused checks.

## Writing the test

```cpp
#include <ApprovalTests.hpp>

TEST_CASE("order_codec - encode new order", "[codec]")
{
  order_codec codec{};
  auto message = codec.encode(make_new_order());
  ApprovalTests::Approvals::verify(message);
}
```

Link the ApprovalTests target into the test binary and include the
Catch2 reporter macro from a single translation unit so failing
approvals surface through the Catch2 runner. A common arrangement is a
tiny `approvals_main.cpp` that contains only:

```cpp
#define APPROVALS_CATCH2_V3
#include "ApprovalTests.hpp"
```

and is added to each test executable's source list alongside the test
files.

Commit approved files next to the test source; gitignore the received
file. On the first run, ApprovalTests writes a received file that the
author reviews and renames to approved.

## Determinism is the contract

ApprovalTests compares bytes. Anything that varies run-to-run -- wall
clocks, monotonic clocks, generated UUIDs, absolute paths, hash-based
iteration order, thread-scheduling artifacts, platform line endings --
will produce flapping failures unless it is normalized before `verify`.

Make the output deterministic at the point of rendering, not by
post-processing the approved file:

- **Sort unordered collections** before serializing. A `std::unordered_map`
  iterated in insertion order today may iterate differently after an
  unrelated change.
- **Replace non-deterministic fields with stable placeholders**:
  `<timestamp>`, `<uuid>`, `<exec_id>`. The placeholder still conveys
  that the field is present and well-formed.
- **Normalize platform-specific separators** (line endings, path
  separators) to a single canonical form.

Centralising scrubbing in one helper keeps mask lists consistent across
tests. Define the helper once and share it across the test files that
need it:

```cpp
const auto scrub = make_scrubber({
  {tags::TransactTime, "transact_time"},
  {tags::CheckSum,     "checksum"},
  {tags::ExecID,       "exec_id"},
  {tags::OrderID,      "order_id"},
});
```

Tests call `scrub(message)` before passing the result to `verify`.

## Aggregating scenarios into one approval

When several closely related scenarios share a shape -- one message per
order action, one rendering per error code -- aggregating them into a
single approved file is usually clearer than one file per case. The
test builds a list of labeled blocks and verifies the concatenation:

```cpp
TEST_CASE("order_codec - encode roundtrip", "[codec][roundtrip]")
{
  order_codec codec{};
  std::vector<std::string> blocks;

  {
    auto message = codec.encode(make_new_order());
    blocks.push_back("# NewOrder - minimal");
    blocks.push_back(scrub(message));
    blocks.push_back("");
  }

  {
    auto message = codec.encode(make_new_order_full());
    blocks.push_back("# NewOrder - full");
    blocks.push_back(scrub(message));
    blocks.push_back("");
  }

  std::stringstream out;
  for (const auto& block : blocks)
  {
    out << block << "\n";
  }

  ApprovalTests::Approvals::verify(out.str());
}
```

The headers act as section markers in the diff so a reviewer can jump
straight to the scenario that changed. Keep the labels stable: renaming
a header reshuffles every line below it and turns a one-line semantic
change into a wall of noise.

A separate file per scenario makes sense when the outputs are
unrelated, when one scenario is much larger than the others, or when
reviewers need to look at one in isolation without scrolling past the
others.

## Reviewing approval diffs

The trap to watch for is approving on autopilot: the diff is too big or
too noisy to read, the author re-approves to make CI green, and a real
regression rides along unnoticed. If you find yourself approving without
understanding every changed line, the test is too broad -- split it or
replace it with assertions that fail with a specific message.

Treat changes to approved files as deliberate, reviewable diffs, the
same as code changes. The diff is the point of the test: it shows, in
human-readable form, exactly how the system's output changed.

For the test to be worth its cost, the reviewer (author and PR
reviewer both) must actually read the diff and confirm that every
changed line is intended. Two practices help:

- **Keep approved files small enough to read.** If a single file runs to
  thousands of lines, split it by scenario or by output type. The point
  of the file is human review; an unreadable file fails that purpose.
- **Pair every diff with a "why".** The commit message or PR description
  says what changed in the output and which code change caused it. A
reviewer comparing approved-file diff to source diff should always be
able to draw the line.

## See also

- `catch2-conventions.md` -- Catch2 registration and assertion conventions.
- `philosophy.md` -- deriving expected behavior from the contract rather than
  the implementation.

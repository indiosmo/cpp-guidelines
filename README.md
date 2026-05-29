# Modern C++ guides

This repository hosts reusable C++ guidelines for modern C++ codebases. It is
meant to be added to a project, often as a submodule, and wrapped by
repo-specific documentation that maps the generic placeholders here to that
project's namespaces, helpers, layout, tooling, and domain examples.

The guides describe how to shape components, test behavior, and investigate
failures without burying the domain model in incidental mechanics. They target
systems where ownership boundaries, error paths, threading, and testability
matter more than isolated language tricks.

## Reading order

The guides are siblings, not a strict sequence. Start where your task starts:
designing a component, writing or repairing tests, or investigating a bug. When
reading broadly, start with the design guide because the testing and debugging
guides reuse its vocabulary.

| Guide | Purpose |
|-------|---------|
| [`cpp-design-principles/`](cpp-design-principles/) | Architecture, types, error handling, performance. |
| [`cpp-testing-principles/`](cpp-testing-principles/) | Test intent, Catch2 conventions, error-path coverage. |
| [`cpp-debugging-principles/`](cpp-debugging-principles/) | Root-cause investigation and structural prevention. |

## Consuming repo layer

A consuming repository should keep these guides generic and add a local mapping
layer next to its own docs. That layer should answer practical questions such
as:

- Which project namespace replaces `lib::`?
- Which headers define result types, strong types, scope guards, logging, time,
  callback wrappers, and test helpers?
- Where do components, adapters, runtime code, and tests live?
- Which local examples show the generic design, testing, and debugging rules in
  real code?

The [`repo-specific-cpp-guidelines`](agent/skills/repo-specific-cpp-guidelines/)
skill bundled in this repository drives that mapping work: it walks the
shared guides and the consuming repo and writes the local mapping folder.
Install the skill into a Claude Code agent that has access to the
consuming repository, then invoke it from there.

## Agent context

The [`agent/`](agent/) folder collects the always-loaded agent
reference ([`cpp-agent-context.md`](agent/cpp-agent-context.md) -- the
rules that shape the code an agent would otherwise write), the
on-demand example companion
([`cpp-agent-examples.md`](agent/cpp-agent-examples.md) -- good/bad
code pairs keyed to the context sections), and the
`repo-specific-cpp-guidelines` skill. Consumers import the context
file unconditionally and load the examples file on demand. Humans
should read the topical guides above instead.

## Conventions

Examples use C++26 unless a section says otherwise; the relevant guide
notes drop-in alternatives for C++20 and C++23, and flags the few
features (for example `std::start_lifetime_as`) that have no clean
older-standard equivalent.

Code fences use `cpp` for C++ snippets, examples use `snake_case`
identifiers, and prose uses US English. Catch2 includes use angle brackets.

Use blank lines to separate phases inside C++ examples. Put a blank line
between setup and a loop, between a multi-line control block and the next
statement, between non-trivial `switch` case groups, and before a final
`return` that follows a loop or branch. Compact one-line `case` arms can stay
adjacent when the layout is easier to scan. Keep tightly coupled guard code
together: a lookup and the `if` that checks it do not need a blank line between
them, and a guard body can keep its log and early `return` together.

Throughout these guides, `lib::` is a placeholder namespace for small in-house
utilities. Substitute the namespace your codebase uses. The examples rely on a
few recurring helpers:

- `lib::result<T>`: the in-domain result type, treated here as the project
  alias for `boost::leaf::result<T>`.
- `std::expected<T, E>`: the boundary result type when callers must see the
  error value in the signature.
- `lib::scope_exit`: a scope guard for rollback and cleanup.
- `lib::inplace_function`: a fixed-capacity callable wrapper.
- `lib::match` and `lib::match_partial`: variant visitation helpers.
- `lib::error` and `lib::new_error`: structured error construction.

## Themes

A handful of ideas recur across the three guides:

- **Domain ownership.** Each component owns its types, errors, and
  invariants. Cross-domain communication happens through adapters that
  translate at the boundary, never through types shared and mutated across
  component boundaries.
- **Types carry the proof.** A well-typed value is, by construction, a valid
  one. Validation happens once at the parser; downstream code receives refined
  types and trusts them.
- **Push effects to the edge.** The functional core stays free of I/O, threads,
  and timers. The imperative shell composes the pure pieces with the
  surrounding side effects. Inner layers therefore test with plain values.
- **Tests encode intent.** Expected values come from the domain, specification,
  contract, or bug report, not from re-running the implementation. A useful
  test fails for a plausible defect and stays readable enough to serve as a
  behavioral example.
- **Compile-time checks where possible, runtime checks where not.**
  Strong types, exhaustive switches, and designated initializers push
  validation into the compiler. Properties the type system cannot express,
  such as state-dependent invariants, fall back to runtime checks.
- **Code encodes a shared model.** The guides aim to keep that model legible
  and refactorable as the team's understanding of the domain evolves.

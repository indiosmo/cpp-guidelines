# Modern C++ guides

These guides describe the C++ style used by this repository: how to shape
components, test behavior, and investigate failures without losing the domain
model in incidental mechanics. They target engineers working on modern C++
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

## Conventions

Examples use C++20 unless a section says otherwise. Code fences use `cpp` for
C++ snippets, examples use `snake_case` identifiers, and prose uses US English.
Catch2 includes use angle brackets.

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
- **Compile-time checks where possible, runtime checks where not.**
  Strong types, exhaustive switches, and designated initializers push
  validation into the compiler. Properties the type system cannot express,
  such as state-dependent invariants, fall back to runtime checks.
- **Code encodes a shared model.** The guides aim to keep that model legible
  and refactorable as the team's understanding of the domain evolves.

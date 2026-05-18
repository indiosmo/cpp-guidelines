# Repo-Specific C++ Guidelines Agent Context

Use this file when creating a project-local mapping layer on top of the shared
C++ guidelines. The shared guidelines are the baseline; the consuming
repository owns the concrete names, headers, tools, and examples.

Do not edit the shared guideline submodule to fit the target project. Add a
small local documentation folder that translates the generic guidance into the
target repo's vocabulary.

## Goal

Create documentation that lets an engineer or coding agent apply the shared
guidelines inside the target repository without guessing:

- which namespace replaces `lib::`;
- which local helpers implement result types, errors, strong types, scope
  guards, fixed strings, variant matching, callback wrappers, clocks, timers,
  logging, and test probes;
- how the repo lays out domains, adapters, runtime wrappers, tests, fixtures,
  and generated or vendor code;
- which real files are the best examples of each pattern;
- which commands verify formatting, static checks, unit tests, sanitizer runs,
  error-code checks, generated code, or other project-specific rules.

## Inputs To Gather

Before writing docs, inspect the target repo for:

- The path to the shared guidelines submodule, such as `docs/cpp-guidelines/`,
  `docs/cpp-guides/`, or another local name.
- Existing agent instructions: `AGENTS.md`, `.claude/`, `.cursor/`, or similar.
- Existing documentation style and location.
- Build system and test runner: CMake, presets, CTest, Catch2, GoogleTest,
  doctest, Qt tests, ApprovalTests, or project scripts.
- Core utility namespaces and headers.
- Component layout under `src/`, `include/`, `lib/`, `test/`, or project-local
  equivalents.
- Domain boundaries, adapter modules, protocol wrappers, vendor wrappers, and
  runtime entry points.
- Error handling conventions: result type, exception boundaries, error-code
  layout, diagnostic context, and top-level handlers.
- Test conventions: factory namespaces, fixtures, probes, singleton guards,
  async waiting helpers, and integration-test boundaries.
- Debugging tools: logging macros, stack traces, sanitizers, bisection scripts,
  replay tools, and deterministic clocks.

Use `rg` and the build files to find these. Prefer real examples already used
by the repo over invented samples.

## Output Shape

Follow the target repo's documentation conventions. When there is no existing
shape, create a folder like:

```text
docs/<repo>-cpp/
  README.md
  design.md
  testing.md
  debugging.md
```

`README.md` should explain how the shared guidelines land in this repo. Include
a placeholder mapping table for recurring names:

| Shared guideline placeholder | Repo-specific mapping | Header or location |
|------------------------------|-----------------------|--------------------|
| `lib::result<T>` | project result type | path to header |
| `lib::error` / `lib::new_error` | project error carrier and construction helper | path to header |
| `lib::strong_type<T, Tag>` | project strong-type helper | path to header |
| `lib::fixed_string<N>` | fixed-capacity string helper, if any | path to header |
| `lib::scope_exit` | scope guard helper | path to header |
| `lib::inplace_function` | fixed-capacity callback wrapper, if any | path to header |
| `lib::match` / `lib::match_partial` | variant visitation helper | path to header |
| boundary result type | `std::expected`, project backport, or local convention | path to header |

Add only mappings that exist or are intentionally absent. If the project has no
local equivalent, say so plainly and point at the preferred existing pattern.
Do not create new utilities just to make the table complete.

`design.md` should map the design guide to the repo:

- component and domain layout;
- public headers versus implementation files;
- adapter naming and placement;
- type ownership and `types.hpp` or equivalent conventions;
- result and error-code layout;
- state-machine, pipeline, runtime, and cross-cutting patterns when present;
- performance-sensitive helper choices;
- one or two canonical examples with links to real files.

`testing.md` should map the testing guide to the repo:

- test target layout and naming;
- test runner and discovery conventions;
- assertion helpers for result or exception paths;
- factory, fixture, probe, mock, provider, clock, and singleton-guard patterns;
- async and GUI testing rules when relevant;
- integration-test boundaries and commands.

`debugging.md` should map the debugging guide to the repo:

- how to reproduce a focused failure;
- logging and diagnostic context;
- stack-trace or error-detail facilities;
- sanitizer, replay, bisection, and tracing commands;
- known flaky-test or shared-state investigation patterns;
- where to document environmental or third-party causes.

If the repo has an `AGENTS.md`, add short pointers from that file to the shared
guidelines and the new local mapping layer. Keep operational commands in the
repo-owned file that agents already load first.

## Writing Rules

- Link to the shared guideline for rationale; write the local file for concrete
  project application.
- Describe patterns, not exhaustive inventories.
- Use one strong example instead of listing every module that follows a rule.
- Use the repo's own names. Do not rename concepts to match the shared guide.
- Keep details next to their source when they change frequently. For example,
  do not copy every CMake target, error code, generated file, or environment
  variable into prose.
- Do not claim a pattern exists until you have found it in the target repo.
- Do not turn the local mapping into a fork of the shared guidelines.

## Completion Checklist

- The local docs link back to the shared design, testing, and debugging guides.
- The placeholder mapping table covers every recurring shared placeholder used
  by the local examples.
- Each topical file points to real target-repo files for its main examples.
- Missing local equivalents are called out without inventing abstractions.
- `AGENTS.md` or the repo's equivalent points agents to the local mapping layer.
- Markdown links resolve from the consuming repo.
- The final answer names the files changed and any commands run to verify them.

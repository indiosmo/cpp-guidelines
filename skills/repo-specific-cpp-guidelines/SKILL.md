---
name: repo-specific-cpp-guidelines
description: Generate a project-local C++ guidelines folder (typically docs/<project>-guidelines/) that maps the shared cpp-guidelines submodule into the project's actual headers, helpers, macros, scripts, and example files. Use this skill whenever the user wants to apply the shared cpp-guidelines submodule to a new project, refresh an existing project mapping layer, port guidelines from one C++ repo to another, or asks "where does the cpp-guidelines stuff land in this codebase". Trigger proactively on mentions of docs/<X>-guidelines/, design.md/testing.md/debugging.md mapping files, repo-specific cpp guides, or any request to translate generic C++ guidelines into project-specific documentation.
---

# Repo-specific C++ guidelines

Use this skill to generate a project's local mapping layer on top of the
shared `cpp-guidelines/` submodule. The shared guides describe generic
principles in a `lib::` placeholder namespace; the local mapping layer
says "which header, which macro, which path, which canonical example in
this codebase".

The job is **translation**, not duplication. If a section of the local
file reads the same as the shared guide with names swapped, the section
is not earning its place.

## Preflight

Before any other work, verify that the shared `cpp-guidelines/`
submodule is present and current in the consuming repository. The skill
depends on reading the shared guides to plan the mapping.

1. **Locate the submodule.** Likely paths are `docs/cpp-guidelines/`,
   `docs/cpp-guides/`, `third_party/cpp-guidelines/`, or a top-level
   `cpp-guidelines/`. Confirm presence by reading
   `.gitmodules`: a `path = ...cpp-guidelines` entry is the signal.
   If `.gitmodules` does not mention it, look for the directory anyway
   in case the project vendored a copy.

2. **Decide what to do based on what you find:**

   - **Present as a submodule, on the tracked commit.** Proceed.
   - **Present as a submodule but pointer is behind upstream.** Run
     `git -C <submodule-path> log --oneline HEAD..origin/main` (or
     equivalent default branch) to show what the project is missing.
     Tell the user the submodule is stale, summarise the missed changes
     in one or two sentences, and ask whether to continue with the
     pinned version or bump first. Wait for the answer.
   - **Present as a plain directory copy (no submodule entry).** Warn
     the user that the project carries a copy rather than a tracked
     submodule, so the local guides may be stale relative to upstream
     and updates will not flow naturally. Ask whether to continue or
     pause to convert the copy into a submodule. Wait for the answer.
   - **Absent.** Abort with a short message: the skill cannot produce a
     mapping without the shared guides as the baseline. Suggest the
     user add the submodule with the upstream URL the project would
     normally use, then re-invoke the skill.

3. **Note the path you found.** Every link in the deliverable that
   reaches into the shared guides must use this path.

Skip steps that do not apply -- if the submodule is on the tracked
commit, move straight to discovery.

## Goal

Produce a `<project>-guidelines/` folder containing up to four files:

| File | Purpose |
|------|---------|
| `README.md` | Reading order, the placeholder mapping table, cross-cutting wiring summary, links to ADRs and the shared guide. |
| `design.md` | Maps `cpp-design-principles/` into project headers, helpers, and example files. |
| `testing.md` | Maps `cpp-testing-principles/` into project test fixtures, factories, probes, and singleton guards. |
| `debugging.md` | Maps `cpp-debugging-principles/` into project logging macros, sanitizer presets, error validators, bisection scripts, and rich-error tooling. |

Skip a topical file if the project genuinely has no equivalent surface
for it -- but say so explicitly in the README rather than emit a stub.

The folder name follows the project's own convention if it has one
already; otherwise pick `<project-short-name>-guidelines/` and place it
under `docs/` (or wherever the project keeps its long-form documentation).

## Inputs to gather (before writing anything)

1. **Shared guides.** From the preflight: the submodule path. Read its
   `README.md` and skim the three sub-tree READMEs so you know what
   shared sections exist and roughly what each says.
2. **Project entry points.** `AGENTS.md`, `CLAUDE.md`, `.cursor/`,
   `.claude/`, root `README.md`, root `INDEX.md`. These often already
   name the project's vocabulary -- harvest it.
3. **Project directory layout.** Look at `src/`, `include/`, `lib/`,
   `test/`. Note the per-component layout convention (boost-stuttering
   `src/<comp>/<comp>/` is common; flat `include/` + `src/` is the
   other common shape). Note where any runtime / threading split lives.
4. **ADRs.** `docs/adr/`, `docs/decisions/`, or similar. Pull in any
   ADR whose decision is observable in the code (logging backend
   choice, result type, error-code policy).
5. **Build and CI.** `CMakeLists.txt`, `CMakePresets.json`, `meson.build`,
   `Makefile`, `.github/workflows/`, `scripts/`. Sanitizer presets,
   error-code validators, format and lint scripts all surface here.
6. **Per-module indexes.** If there's a `src/<comp>/INDEX.md` (or
   equivalent), it usually lists public vs internal headers and the key
   types -- gold for finding canonical examples.

Use `rg` / `grep` and direct reads. Do not hallucinate paths -- every
link in the output must resolve to a real file.

## Discovery: walk the shared guide, ask one question per section

For each section of the shared guide, ask: **"What concrete project
symbol, file, macro, or script realizes this?"** Three possible
answers:

1. **Direct mapping exists.** Project has its own `lib::result<T>`
   equivalent (`mil::result<T>`, `core::result<T>`, `std::expected<T, E>`,
   etc). Record the symbol + the header that defines it.
2. **Mapping exists but is structured differently.** Project owns the
   concept but splits it across more files or wraps it differently
   (e.g. a three-file domain triad `error_code.hpp + errors.hpp +
   error_handlers.hpp` where the shared guide describes one of those).
   Record the shape, not just the symbol.
3. **No equivalent.** The project genuinely lacks the pattern (e.g. no
   inplace-function wrapper, no scope-guard library). Note it
   explicitly in the placeholder table as "n/a; project uses X instead"
   or "absent; prefer Y pattern". Do not invent.

## Discovery: walk the project for patterns the shared guide does NOT name

The second source of content is the project's own recurring patterns.
Look for shapes the shared guide does not cover but that recur often
enough to deserve a name:

- Adapter naming conventions (`<dom1>_<dom2>/`,
  `adapters/<external>/`, `clients/<vendor>/`).
- Wiring helpers (functions that bulk-connect callback fields, no-op
  wirings for tests, proxy stages).
- Cross-cutting service registration patterns (variant-backed globals,
  service locator, dependency-injection roots).
- Error-code numeric allocation schemes (`601xxx` for `aor`, etc.) and
  the validator script that enforces them.
- Macros that abstract recurring expressions (`MIL_PLUCK`,
  `MIL_CONTINUE_ON_ERROR`, `MIL_REQUIRE_LEAF`).
- Per-domain `types.hpp` conventions plus any local namespace-alias
  shorthand (`namespace at = aor::types;`).
- Per-module navigation files (root `INDEX.md` + per-module `INDEX.md`
  with public/internal tagging).
- JSON-config-driven runtime composition pipelines.
- Test-side singleton-sandboxing helpers.

`rg` for recurring tokens. Read a handful of the most-cited headers from
`INDEX.md`. Skim `AGENTS.md` -- if a pattern is repeated there as a
warning ("non-defaulted callback type X asserts on destruction"), that
is signal it bites often and belongs in the local guide.

## Writing discipline: the restate-vs-map test

Before keeping a section, paragraph, or example in the local guide,
ask:

> If I replaced the project names with `lib::` placeholders, would this
> example differ from what the shared guide already says?

If no, delete it. The shared guide is loaded as agent context;
restating its content burns the reader's attention budget and creates
two sources of truth that will drift.

Concrete tells that something fails the restate-vs-map test:

- A code example that demonstrates a rule the shared guide already
  demonstrates, with no project-specific helper or recurring `N`
  choice.
- A paragraph of prose explaining *why* a pattern is good (the shared
  guide owns the rationale; the local file is the index, not the
  textbook).
- A repeated definition of what `lib::strong_type`, `BOOST_LEAF_CHECK`,
  or `std::variant` do.

Tells that something *passes* the test:

- A line of the form "project symbol X realizes shared concept Y;
  defined in `path/to/header.hpp`".
- A pattern the project follows that the shared guide does not name
  (adapter codec directories, the `wire_noop` recipe, the
  `error_code.hpp + errors.hpp + error_handlers.hpp` triad).
- A non-obvious project consequence of following a shared rule (e.g.
  "non-defaulted callbacks assert on destruction, so tests must wire
  every callback or use the noop helper").

## Writing discipline: one canonical example per pattern

The shared writing rules ban listing every module that follows a rule.
Apply the same rule to code snippets inside a section: pick the single
file most worth reading and link there. Do not paste two or three
near-identical snippets to illustrate the same shape -- the reader will
copy the first one and never reach the second.

When two examples really do illustrate different things (e.g. one
shows the abstract interface, the other shows the per-counterparty
implementation), keep both but say what each adds.

## Writing discipline: the placeholder mapping table lives only in README.md

The **placeholder mapping table** -- the one whose rows pair a shared
`lib::*` placeholder with a project symbol and its defining header --
belongs in **one** place: `README.md`. Topical files (`design.md`,
`testing.md`, `debugging.md`) reference symbols freely and trust the
reader to consult the table for header paths.

This matters because mapping rows are the highest-velocity drift hazard
in the deliverable -- a header gets moved, the row updates, and if the
same row is duplicated three files away it silently rots.

When `design.md` needs to name a macro or symbol that already lives in
the table, write the symbol name plain (`mil::strong_type`, `MIL_PLUCK`)
without re-linking to the defining header. Add a header link only when
the section is the canonical reading-order entry for that symbol.

The rule is narrow. **Other project-internal tables are fine in topical
files.** Tables of `fixed_string<N>` size choices, error-code numeric
prefix allocations, sanitizer presets, test-target layout, variant
singleton alternatives, and similar project-only information belong
next to the section that uses them and do not need to migrate to the
README. The signal that a table is the placeholder mapping is the
presence of `lib::*` (or "shared placeholder") in the left column --
not the presence of project symbols on the right.

## File shape: README.md

Opens with one paragraph naming what this folder is and the
relationship to the shared submodule. Then:

1. **Reading order table.** One row per topical file (`design.md`,
   `testing.md`, `debugging.md`), with a "use it when" hook.
2. **Placeholder mapping table.** Three columns: shared placeholder,
   project symbol, defining header. Add rows for every recurring
   placeholder used by the local examples. After the table, a short
   "absent / different" list for placeholders that have no project
   equivalent.
3. **Cross-cutting wiring summary.** Three to five bullets naming the
   conventions that shape almost every file in `src/` (domain
   ownership, pipeline callback wiring, cross-cutting services,
   runtime split). Each bullet is one or two sentences -- the topical
   files carry the detail.
4. **Build, test, verification commands.** Point to the relevant skill
   or script for each (build presets, sanitizer wrapper, error-code
   validator, flaky-test bisector). Do not list every preset name --
   `CMakePresets.json` is the source of truth.
5. **Relation to other docs.** Short bullets pointing at `AGENTS.md`,
   `INDEX.md`, the shared submodule, ADRs.

## File shape: design.md

Opens with one paragraph naming what the file does and a pointer at
the README's placeholder table. Then one section per project-specific
design concept, in this rough order:

- Component and domain layout (the project's per-component directory
  shape, where runtime wrappers live, how adapter modules are named).
- The `types.hpp` convention (or equivalent vocabulary file) and which
  strong-type helper / `N` choices recur.
- Error handling (the file triad if any, numeric prefix scheme,
  validator script, catch-macro family, `make_error` shortcut
  signatures).
- Pipelines and wiring (the project's wiring helper family if any, and
  the non-obvious consequences of the chosen callback storage type).
- Runtime layer wrappers (the canonical marshalled-source / sink shape,
  the message-queue typedef with its capacity tuning).
- Cross-cutting singletons (the variant-backed globals, how `main`
  composes them, how tests sandbox them).
- Adapter pattern (the recurring "abstract interface + per-X impls +
  runtime layer" recipe, plus any snapshot-type trick).
- Hot-path helpers (recurring `N` choices, pre-reserved containers,
  callback storage sizing, project annotations like "do not adopt X
  -- broken").
- Navigation file convention (per-module `INDEX.md` and how to refresh
  it).

Skip a section if the project genuinely has no instance of it. Add a
section if the project has a recurring pattern not on this list.

## File shape: testing.md

Same opening shape. One section per project-specific testing concept:

- Test target layout and naming (Catch2, gtest, doctest -- whichever),
  including how tests mirror `src/`.
- Result-aware assertion helpers (`MIL_REQUIRE_LEAF`,
  `mil::testing::require_error`, equivalents).
- Factory / fixture / probe conventions (where they live, naming).
- Singleton-sandboxing RAII guards (for tests that touch a
  variant-backed global).
- Deterministic clock and time-control helpers (`mock_clock`, etc).
- Async waiting helpers (predicate-wait functions, timeout helpers).
- Integration-test boundaries (which directories or tags signal "this
  spins real threads / sockets / vendor engines"; how to run only the
  fast tests).
- Approval-test conventions if present (scrubbers, output directory
  layout, review-tool invocation).
- The "ensure all callbacks are wired in tests" recipe, if relevant.

## File shape: debugging.md

Same opening shape. One section per project-specific debugging
concept:

- Logging macros and how to enable trace builds.
- Stack traces and rich error details (full-details helpers, stacktrace
  toggle helpers, third-party trace libraries in use).
- Sanitizer presets and the wrapper script that runs them.
- Flaky-test bisection helper / state-pollution detection.
- Error-code validator (`eccheck.sh` or equivalent) -- what it
  enforces.
- Replay or capture tools, if the project has them.
- Where to record environmental or vendor-library debugging notes.

## Self-review pass (before declaring done)

Walk the draft top to bottom and check each section against this list.
Any "yes" is a defect to fix.

- Could I delete this section and lose nothing the shared guide doesn't
  cover? (Then delete it.)
- Does this code example exist in the shared guide with different
  names? (Then delete it.)
- Does this paragraph re-explain *why* a pattern is good? (Then trim
  to one sentence pointing at the shared guide for rationale.)
- Does this section paste two near-identical code snippets to show the
  same shape? (Then keep one, link the other.)
- Does this row appear in the README mapping table and also in this
  topical file? (Then delete the topical-file restatement.)
- Does this link resolve? (Run `ls` on every path you wrote.)
- Did I cite a real example file, not a generic shape? (Open it and
  confirm the cited symbol is at the cited line.)

Also: if the project has an `AGENTS.md`, `CLAUDE.md`, or equivalent
agent-entry doc, add a short pointer from there into the new folder.
The local mapping is most useful when agents can find it on the first
pass.

## Anti-patterns

- Writing prose that explains why strong types are good. The shared
  guide owns rationale.
- Pasting the shared guide's enum-switch / variant-match / state-machine
  examples with project names. The reader has both files open.
- Inventing a project utility because the shared guide mentions one and
  the project lacks an equivalent. Say it is absent and move on.
- Listing every adapter, every codec, every error code. Generalize and
  point at the directory.
- Front-loading topical files with directory trees. The root `INDEX.md`
  is the project map.
- Duplicating the placeholder mapping table across topical files.

## Tone

Plain, direct, present tense. No marketing voice. No glyphs. Imperative
mood for procedures, declarative for descriptions. Match the project's
existing documentation voice -- if the project uses British spelling,
match it; if it writes `std::` qualifiers everywhere, match that too.

The local guide is an index. The shared guide is the textbook. Write
accordingly.

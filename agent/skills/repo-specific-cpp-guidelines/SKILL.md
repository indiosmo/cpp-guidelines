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

## Writing discipline: the three-bucket filter

Every section, paragraph, and bullet in a topical file
(`design.md`, `testing.md`, `debugging.md`) must fit one of three
buckets. If a candidate sentence does not fit any bucket, delete it.

1. **Additions** -- something the project has that the shared guide
   does not name at all. Examples: a numeric-prefix scheme for error
   codes plus the validator script that enforces it; a wiring-helper
   family that bulk-connects callback fields; an adapter recipe of
   "abstract codec + per-counterparty impl + runtime layer"; a JSON
   config + PFR-reflection macro pair.
2. **Deltas** -- something the project does differently from the
   shared guide's default shape. Examples: the shared guide describes
   a single `errors.hpp`, but the project splits into a triad
   (`error_code.hpp + errors.hpp + error_handlers.hpp`); the shared
   guide names a generic `lib::scope_exit`, but the project uses
   `ricab/scope_guard` directly with no wrapper.
3. **Consequences** -- a non-obvious project-specific outcome of
   following a shared rule. Examples: callback fields are
   `mil::inplace_function`, which is non-defaulted, so tests must
   wire every callback or trip a destruction assert; handler order
   in `std::tuple_cat` matters because `boost::leaf` walks the tuple
   linearly.

If a candidate paragraph reads like "the shared guide says X; in
this project, X looks like Y", it is almost certainly restatement.
Strip the "shared guide says X" half. The reader has the shared
guide loaded.

### Tells of restatement

- A paragraph framing or motivating a rule before stating the
  project's realisation. ("The shared guide pushes strong typing,
  exhaustive enum switches, ...") Delete the framing; jump to the
  project specifics.
- A code example that demonstrates a shared-guide rule with no
  project-specific helper, `N` choice, or gotcha.
- A description of what `lib::strong_type`, `lib::match`,
  `BOOST_LEAF_CHECK`, or `std::variant` does.
- A "reserve this pattern for genuine X" sentence (rationale belongs
  in the shared guide).
- A sentence that lists what something is *not* responsible for
  ("this is not a thread-safe component"), unless the boundary is
  load-bearing and non-obvious from the type.

### Worked example

A pipelines section that fails the test:

> ## Pipelines and callbacks
>
> The shared guide describes pipeline stages as components with
> `on_*` callback fields and `send` overloads. The canonical
> realisation lives in `src/aor/aor/pipeline_stage.hpp`. A stage
> exposes:
>
> - one virtual `send(Message&&)` per message type;
> - one public `mil::inplace_function<void(Message&&)>` callback
>   field per message type.
>
> Wiring near `main` assigns the callbacks. Non-obvious consequence:
> these callbacks are non-defaulted and assert on destruction in
> debug builds...

The first three paragraphs restate the shared guide's pipeline
shape; only the consequence-bullet earns its place. The trimmed
version reads:

> ## Pipelines and callback wiring
>
> Pipeline interfaces follow the shared on_* + send shape. Project
> additions live in `src/aor/aor/wiring.hpp`: `wire_requests`,
> `wire_responses`, `wire`, `wire_proxy`.
>
> **Non-obvious consequence.** Callback fields are
> `mil::inplace_function`, which is non-defaulted and asserts on
> destruction if it was never assigned. `scripts/wirecheck.sh`
> catches most production omissions; tests must wire noops by hand.

One sentence pointing at the shared shape ("follow the shared on_* +
send shape"), then the additions and the consequence. Everything
else is gone.

### Section-length smell

If a topical-file section runs longer than five or six short
sentences or bullets, suspect restatement. The local guide is an
index; a section that needs two paragraphs of prose is almost
always doing the shared guide's job too.

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

## Writing procedure: extract bullets first, then prose

Do not draft prose top to bottom. Each topical file is built in three
passes:

1. **Topic walk.** Go through the shared guide's matching tree
   (`cpp-design-principles/` for `design.md`, etc.) one file at a
   time. For each shared file, ask only the three-bucket questions:
   does the project add anything here, do anything differently, or
   suffer any non-obvious consequence? Capture each answer as a
   single bullet with the concrete symbol, path, or script attached.
   If a shared file yields zero bullets, write nothing for it.

2. **Group into sections.** Cluster the bullets that share a topic
   (error handling, pipelines, runtime, ...). Each cluster becomes
   one section. A cluster with one bullet is a fine section -- write
   the bullet plus its routing sentence and stop. A topic that
   produced zero bullets does not appear in the file at all.

3. **Prose only as needed.** For each section, write the minimum
   routing prose to introduce the bullets -- usually one sentence.
   No framing, no motivation, no rationale. If the routing sentence
   restates the shared guide, delete it; the bullets are enough.

The order of sections in the final file follows whichever cluster
the reader is most likely to need first; the topic walk in pass 1 is
discovery scaffolding, not the output order.

## What the topical files actually contain

The three topical files are not "everything about design / testing /
debugging in this project". They are the project's deltas, additions,
and consequences on top of the shared guide. Anything the shared
guide already covers without project specifics belongs to the shared
guide.

Common categories of content (use as a checklist during the topic
walk; do not turn into mandatory sections):

**design.md typically contains, when the project has them:**

- the component directory layout (boost-stuttering, flat, or
  whatever the project uses), including where runtime wrappers and
  test helpers live, and how adapter modules are named;
- recurring `mil::fixed_string<N>` / `mil::inplace_function<Sig, N>`
  size tables;
- the X-macro / reflection-macro tricks the project uses
  (`MIL_AUTO_JSON_PFR_NAMESPACE`, X-macro composite enums);
- the error-handling triad (`error_code.hpp` + `errors.hpp` +
  `error_handlers.hpp`), the numeric-prefix scheme, the catch-macro
  family, the `make_error` shortcut, the validator script;
- the wiring-helper family (`wire`, `wire_proxy`, `wire_noop`) and
  its consequence: callbacks that assert on destruction;
- the runtime layer split (functional core + `<domain>/runtime/`),
  marshalled adapters, message-queue typedefs and their capacity
  tuning;
- the cross-cutting singleton pattern realised by the project's
  variant-backed globals, plus how tests sandbox them;
- the adapter recipe (abstract interface + per-X impls + runtime
  layer);
- hot-path helper choices and "do not adopt X -- broken" notes;
- per-module `INDEX.md` / `README.md` conventions.

**testing.md typically contains:**

- the test-target layout (mirroring `src/`), the test framework
  invocation glue (`catch_discover_tests`, etc.), and the tag
  convention;
- result-aware assertion helpers
  (`mil::testing::require_error`, equivalents);
- where factories, fixtures, and probes live and the naming pattern;
- singleton-sandboxing RAII guards;
- the deterministic-clock helper, plus the predicate-wait family;
- integration-test boundaries (which tags or directories signal
  "this boots a real vendor engine / socket / thread");
- approval-test scaffolding (scrubbers, output directory layout);
- the "wire every callback in tests too" recipe.

**debugging.md typically contains:**

- the logging-macro family, level-toggle, throttle variants, and
  logger backends;
- stack-trace integration (`cpptrace`, `boost::stacktrace`),
  always-on stacktrace toggles, full-details helpers;
- the sanitizer-preset table and suppression file locations;
- flaky-test bisection invocation and reproducibility tools;
- error-code validators and wiring validators;
- domain-specific replay or capture tools;
- where to look first when investigating a class of bug.

These are categories to scan during discovery, not headings to
guarantee. A project that has no flaky-test bisector simply does
not get a flaky-test section.

## Self-review pass (before declaring done)

Walk every section of every topical file and apply the three-bucket
filter sentence by sentence. For each sentence, classify it:

- (A) an **addition** -- project pattern the shared guide does not name;
- (D) a **delta** -- project does this differently from the shared default;
- (C) a **consequence** -- non-obvious outcome of following a shared rule;
- (R) a routing sentence pointing at a file, symbol, or section
      (allowed once or twice per section to introduce bullets);
- (X) anything else (framing, rationale, restatement, motivation).

Every (X) is a defect. Delete it. If a section is all (R)s and (X)s
after the pass, the section has no content and should disappear.

Then run the mechanical checks:

- Does this link resolve? (`ls` every path you wrote.)
- Did I cite a real example file, not a generic shape? (Open it and
  confirm the symbol is at the cited line.)
- Does this row appear in the README mapping table and also in a
  topical file? (Delete the topical-file restatement.)
- Is any section longer than ~6 short sentences or bullets? (Suspect
  restatement; rerun the bucket filter on that section.)

If the project has an `AGENTS.md`, `CLAUDE.md`, or equivalent
agent-entry doc, add a short pointer from there into the new folder.
The local mapping is most useful when agents can find it on the first
pass.

## Anti-patterns

- Writing framing or rationale prose before getting to the
  project-specific bullet. The reader has the shared guide loaded;
  jump straight to additions, deltas, and consequences.
- Drafting topical files top to bottom in prose rather than
  extracting bullets first. The bullet-first procedure exists because
  prose-first drafting reliably produces restatement padding.
- Treating the content categories above as a checklist of required
  sections. A project that has no flaky-test bisector does not get a
  flaky-test section.
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

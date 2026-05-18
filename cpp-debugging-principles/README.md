# C++ Debugging Principles

A guide to investigating bugs, test failures, and unexpected behavior in a way
that finds the actual cause instead of papering over the symptom. The
technique is general; the examples are C++.

A companion guide, `cpp-testing-principles`, describes how to write tests
that pin behavior down once a bug has been understood.

**Core principle:** no fix without root-cause investigation. A change
that makes the test turn green is not the same as a change that fixes the
bug; if the root cause is still there, the next failure from the same source
is already on its way. Systematic debugging is faster than guess-and-check
thrashing, because each round of guessing costs build time, test time, and
the cognitive load of holding the last several attempts in mind.

## The four phases

Treat debugging as four ordered phases. If a fix attempt fails, the response
is not another fix -- it is a return to Phase 1 with the evidence the
attempt just produced.

### Phase 1: root-cause investigation

Before any code changes, find out what is actually happening.

**Read the error carefully.** Stack traces, line numbers, and error codes
often contain the answer. A common failure mode is to fix a hypothetical
bug while the actual error message names a different problem two lines
down. Read the whole trace, including the parts that look like noise.

**Reproduce it consistently.** Establish the exact steps that trigger the
bug. If the bug is intermittent, that is itself a clue -- a race condition,
an ordering dependency, an environmental factor -- but a fix proposed
against a failure you cannot reproduce is a guess. The reproduction is also
what you will use later to verify the fix actually worked.

**Look at recent changes.** What changed in the codebase, the environment,
the dependencies, or the build just before the bug appeared? `git log`,
`git bisect`, and the diff against the last known good state are usually
more efficient than reading the failing code cold. A bug that appeared
recently has a small set of recent changes that could have caused it; that
set is the first place to look.

For the full procedure -- logging at component boundaries, tracing data flow backward, and bisecting test pollution -- see `root-cause-tracing.md`.

### Phase 2: pattern analysis

Once you understand the symptom, look for patterns rather than diving
straight into a fix.

**Find working examples.** Most codebases contain similar code that already
works. If you are debugging a parser, find a parser that works. If you are
debugging a thread pool, find a thread pool that works. The difference
between working and broken is usually a short list you can enumerate.

**Read references completely.** If the code implements a published pattern
-- an RFC, a specification, a library's idiomatic usage -- read the
reference all the way through before assuming the bug. "I have the rough
shape of the pattern" is how partial implementations creep in: the bug is
in the part you skimmed.

**List every difference.** Compare working to broken and write down every
difference, however irrelevant it seems. The bug is usually in something
you nearly dismissed because "that can't matter".

**Understand dependencies.** What other components, configuration, or
environment does the code under investigation need? An assumption about an
implicit dependency -- a thread already running, a directory already
present, an environment variable already set -- is one of the most common
sources of bugs that reproduce only in certain contexts.

### Phase 3: hypothesis and testing

With evidence and pattern in hand, form a single, specific hypothesis and
test it.

**State the hypothesis explicitly.** "I think the cache holds a stale value
because `invalidate()` is not called on the failure path." Write it down.
A vague hypothesis -- "something in the cache is wrong" -- cannot be tested
or disproved, so it cannot move you forward.

**Test minimally.** Make the smallest change that proves or disproves the
hypothesis. One variable at a time. The point of the test is to learn
whether the hypothesis is correct, not to fix the bug yet. A test that
changes three things tells you nothing useful when the result comes back.

**Verify before moving on.** If the test confirms the hypothesis, advance
to Phase 4. If it disproves it, form a *new* hypothesis -- do not stack a
second guess on top of the first. A five-hypothesis stack means none of them was tested.

When understanding is too thin to form a hypothesis, naming that
explicitly -- "I don't yet understand why X happens" -- is more useful
than an unjustified guess. The next step is further investigation, a
conversation, or a re-read of the relevant code.

### Phase 4: implementation

Only now, with the root cause identified, do you change production code.

**Write a failing test first.** The simplest reproduction you can build.
Until the test fails for the right reason, you do not have a precise
description of the bug, and you cannot tell whether the fix worked. The
test is the pinned contract that defines what "fixed" means. See
`../cpp-testing-principles/test-patterns.md` and
`../cpp-testing-principles/error-path-testing.md` for the test shapes.

**Make one change.** Fix the root cause and nothing else. Bundling
cleanups, renames, or unrelated improvements into the same commit turns
the next regression's `git bisect` into a multi-issue debugging session.
When the fix exposes a bug class, use `defense-in-depth.md` to close the
other paths that could reintroduce it.

**Verify the fix and the suite.** The failing test now passes. The rest of
the test suite still passes. The original symptom no longer reproduces by
hand. If a fix is "done" but the suite is not green, the fix is not done.

**If the fix does not work, return to Phase 1.** Do not stack a second fix
on top of the first. Each failed attempt is evidence -- new information
that sharpens the hypothesis. Use it.

## When the root cause is not a code defect

When each fix uncovers a new problem in a different place, the pattern being
implemented is fighting the design. The next step is a conversation about
the design, not another fix.

Sometimes investigation shows the cause is environmental, timing-dependent,
or external -- a third-party service, an OS scheduling quirk, a hardware
fault. In that case:

1. Document what was investigated and ruled out.
2. Implement an appropriate response -- a retry with backoff, a clear
   timeout, an explicit error message, a fallback path.
3. Add monitoring or logging so the next occurrence comes with more data.

## Navigation

| File                         | Covers |
|------------------------------|--------|
| `root-cause-tracing.md`      | Tracing a bug backward through the call stack to its root cause, including stack-trace instrumentation and test-pollution bisection. |
| `defense-in-depth.md`        | Adding validation at every layer the data passes through, so the same bug class becomes structurally impossible to reintroduce. |

When investigating flaky tests, the predicate-based wait pattern in
`../cpp-testing-principles/condition-based-waiting.md` removes the most
common timing failure.

Tooling still matters: sanitizers, debuggers, and stack traces can supply the
evidence for Phase 1. This guide focuses on the investigation loop rather
than on sanitizer or debugger setup.

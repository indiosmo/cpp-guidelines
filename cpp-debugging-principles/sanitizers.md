# Sanitizer presets

Sanitizers turn a class of runtime defect into a deterministic test
failure. Their cost on a build is significant, so the project ships a
preset per sanitizer and runs each preset on its own schedule rather
than enabling them together by default.

## Ship a preset per sanitizer

A project that uses sanitizers ships build presets for `asan` (with
UBSan riding along), `tsan`, and `msan` at minimum. `filc` (Fil-C) is
worth a preset when the toolchain is available. Each preset wires the
sanitizer flag and points the runtime at the project's suppression file
via the matching environment variable -- `ASAN_OPTIONS`,
`TSAN_OPTIONS`, `MSAN_OPTIONS`, `UBSAN_OPTIONS` -- so a fresh shell does
not need manual export to reproduce a sanitizer run.

Each preset owns its own build directory. A sanitizer build and a
release build cannot share object files; sharing a build root produces
linker errors at best and undefined behavior at worst.

## Suppression files live next to the build, version-controlled

Each sanitizer gets its own suppression file, checked into the
repository alongside the build configuration:

- `asan_suppressions.txt`
- `lsan_suppressions.txt`
- `tsan_suppressions.txt`
- `msan_suppressions.txt`
- `ubsan_suppressions.txt`

UBSan suppressions are consumed when UBSan is enabled alongside ASan;
they have their own file even when the build pulls them in through the
ASan preset.

Suppression files belong with the build configuration, not in a
developer's shell profile. A suppression that only takes effect when one
person runs the build is a defect that fails for everyone else.

## Triage before suppressing

A sanitizer report is evidence, not noise. Add a suppression only after
the underlying defect is understood and ownership of the unfixable frame
is documented in the suppression file as a comment -- a vendor library
that intentionally races on a metric counter, a system call whose error
path is reported as a leak by design.

A suppression with no explanation is a deferred bug. Six months later,
no one remembers whether the suppressed frame was a real defect or a
tolerated quirk; the safe assumption is that it was a defect, which
means the suppression should be removed and the report re-triaged. A
short comment alongside each suppression makes the next triage cheap.

## ThreadSanitizer is special-cased

TSan reports need root-cause triage before suppression more often than
the other sanitizers do, because the report points at the observed
interleaving rather than at the actual race. The project's TSan policy
document covers the known third-party noise and the shape of a triaged
TSan suppression entry. Do not pre-emptively silence vendor warnings;
the silenced report is the one that would have caught the same race in
project code.

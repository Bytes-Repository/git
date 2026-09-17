# Where the tests actually are

hgit's tests live in `src/`, next to the modules they exercise
(`src/oid_test.h#`, `src/object_test.h#`, `src/repository_workflow_test.h#`),
not in this directory.

Why: an H# `mod some_module` statement resolves `some_module.h#` relative
to the *file declaring it*. A test file physically sitting in `tests/`
would need to reach back into `../src/` for the modules it wants to
test, which isn't how the language's module resolution works today.
`bytes test`'s own discovery logic special-cases this exact situation —
see `bytes`' `src/test_runner.h#`, `collect_test_files()` — by scanning
`src/` for `#[test]` blocks in addition to files under `tests/`/named
`*test*`/`*spec*`, so putting tests directly in `src/` is the
already-supported, already-working path rather than a workaround.

Run them with:

```sh
bytes test
```

This directory is kept (rather than removed) purely as the
conventional place a consumer of this library would look first; see
its note above for where to actually find things.

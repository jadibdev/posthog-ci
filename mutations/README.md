# Mutation checks

A mutation check proves that a test catches the bug its fix prevents: revert the fix, run the test, and watch it fail.
`cohort-replace-quick.yml` runs a whole directory of them in one job, on one environment, without any throwaway
branches.

## Format

One directory per round, for example `mutations/r7/`. Each mutation is a pair of files with the same name:

- **`<name>.patch`:** a unified diff, from the repo root (`a/…` and `b/…` paths), that reverts or breaks one fix.
  `git apply` must accept it against the ref you run.
- **`<name>.tests`:** the pytest targets that must fail with the patch applied, one per line. Lines starting with `#`
  are comments. An empty list is rejected, because it would run every test in the repo.

Names sort into run order, so prefix them with a number: `01-repair-no-page-check.patch`.

## Making a patch

Edit the file in your checkout to introduce the bug, then save the diff and restore the file:

```bash
git diff -- path/to/file.py > ../posthog-ci/mutations/r7/01-name.patch
git checkout -- path/to/file.py
```

Check that it applies to the ref first: `git apply --check ../posthog-ci/mutations/r7/01-name.patch`.

## Running

```bash
gh workflow run cohort-replace-quick.yml -R jadibdev/posthog-ci -f ref=<branch or sha> -f mutations=mutations/r7
```

The job summary has a table: one row per mutation, with its result and the tests that failed.

- **caught:** the listed tests failed (pytest exited 1). That's the pass case.
- **survived:** the tests passed with the bug in place, so they don't guard the fix.
- **error, does not apply, or empty tests file:** the check proved nothing. Fix the patch or the list.

The job fails unless every mutation is caught. `validation/` holds one known mutation that must be caught. Use it to
check the workflow itself after changing it.

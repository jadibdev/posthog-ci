# posthog-ci

Runs checks for branches of [jadibdev/posthog](https://github.com/jadibdev/posthog) on GitHub-hosted runners.

The workflows here check out the requested ref of the fork and test it. They live in their own repository so that enabling Actions doesn't also run the upstream PostHog workflows that the fork carries.

## Cohort replace checks

`.github/workflows/cohort-replace-checks.yml` validates the static cohort replace work ([PostHog/posthog#121486](https://github.com/PostHog/posthog/issues/121486)). It builds the environment the way PostHog's backend CI does, and has two jobs:

- **Lint:** checks each of the branch's commits for undefined, unused or redefined names, then runs ruff and mypy on the files the branch changes.
- **Tests:** migrates a fresh database, confirms there are no missing or conflicting migrations, then runs the cohort, task and API suites, plus the suites for files upstream also changed.

Run it from the Actions tab, or:

```bash
gh workflow run cohort-replace-checks.yml -R jadibdev/posthog-ci -f ref=<branch or full sha> -f commits=<number of branch commits>
```

## Quick checks and mutation checks

`.github/workflows/cohort-replace-quick.yml` sets up the same environment once, in one job. Then it runs any pytest targets you name, a directory of mutation checks (see `mutations/README.md`), or both:

```bash
gh workflow run cohort-replace-quick.yml -R jadibdev/posthog-ci -f ref=<branch or sha> -f targets="path/to/test_file.py"
gh workflow run cohort-replace-quick.yml -R jadibdev/posthog-ci -f ref=<branch or sha> -f mutations=mutations/r7
```

## Where the time goes, and which run to start

- **Migrating the test database takes about 17 minutes of each test job's 27.** The quick workflow and targeted runs cache the migrated database, keyed by every file under any `migrations/` folder, `uv.lock` and `docker-compose.dev.yml`. The first job after any of those changes migrates from scratch and saves the dump; later jobs restore it in seconds. The dump is taken right after migrating, before any test runs.
- **A broad run never uses the cache.** It's the gate before a reviewer-visible push, so every job migrates from scratch on a fresh runner, as before. Everything else in every run is fresh too: the runner, the checkout, the service containers, the Python dependencies, ClickHouse and the persons database. The cache only shortens iteration. A migration whose result depends on app code outside `migrations/` could make a cached run test a stale schema, and the broad run would still catch it.
- **Caches made on `main` are shared** with every run; caches made on another branch of this repo aren't.
- **Runs of `cohort-replace-checks.yml` on the same ref queue behind each other** (its concurrency group). The broad groups already include the targeted suites, so never start a targeted run next to a broad one.
- **Which to start:**
  - To check a fix: the quick workflow, with the affected test files as targets.
  - To prove tests catch their bugs: the quick workflow, with a mutations directory. One job, and no throwaway branches.
  - Before a reviewer-visible push: one `cohort-replace-checks.yml` run with `broad=true`, plus the other options the change needs.

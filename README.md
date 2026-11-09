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

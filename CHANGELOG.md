# CHANGELOG

## Unreleased

- Add `version` output to tagandrelease.yml action
- Add `on.workflow_call.secrets.github-token` to tagandrelease.yml action
- Add `changedonlychangelog.yml` action
- Add `set-requires-testing-status.yml` action
- Add `append-testing-sheet-entry.yml` action
- Add PR number to data inserted in `append-testing-sheet-entry.yml`
- Use PR number to match rows in `set-requires-testing-status.yml`
- Add `platforms` input argument to build-and-deploy-ecr-image.yml
- Add `runner` input argument to build-and-deploy-ecr-image.yml
- Replace deprecated `::set-output` with `$GITHUB_OUTPUT` and upgrade all third-party actions to latest versions (SHA-pinned)
- Move `${{ }}` expressions out of `run:` blocks into `env:` to prevent shell injection

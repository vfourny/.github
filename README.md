# .github

Shared GitHub configuration for all `vfourny` repositories:

- **Reusable workflows** (build, release, PR checks)
- **Default community files** (issue templates, PR template, `CONTRIBUTING.md`, `SECURITY.md`), applied to every repository that doesn't define its own
- **Shared Renovate preset** (`default.json`)

## Versioning

This repository is released with [semantic-release](https://semantic-release.gitbook.io/): merging a PR on `main` creates a `vX.Y.Z` tag and a GitHub release when its Conventional Commits title calls for one (`fix:` → patch, `feat:` → minor, `!` or `BREAKING CHANGE` → major). Other types (`docs:`, `chore:`, `ci:`, `test:`…) don't publish a release.

**Always reference a release, never `@main`**, so a change here can't break your repositories. With the Renovate preset below, references are pinned to a commit SHA and updated through PRs (patch/minor automerged, majors reviewed manually):

```yaml
uses: vfourny/.github/.github/workflows/build.yml@v2.0.0
```

## Reusable workflows

### `build.yml`

Installs dependencies, checks formatting, lint and types, builds, runs the tests, and uploads `dist` as an artifact.

| Input                 | Type    | Default | Description                                               |
| --------------------- | ------- | ------- | --------------------------------------------------------- |
| `node_version`        | string  | `lts/*` | Node.js version passed to `actions/setup-node`            |
| `is_build`            | boolean | `true`  | Run `npm run build`                                       |
| `is_build_upload`     | boolean | `true`  | Upload `dist` as the `dist` artifact (requires `is_build`) |
| `is_type_checked`     | boolean | `true`  | Run `npm run ts:check`                                    |
| `is_tested`           | boolean | `true`  | Run `npm run test`                                        |

Required npm scripts: `format:check`, `lint`, plus `build`, `ts:check` and `test` depending on the inputs.

### `release.yml`

Runs [semantic-release](https://semantic-release.gitbook.io/) with a GitHub App token. If `is_build` is set, it reuses the `dist` artifact from `build.yml` (same workflow run).

| Input          | Type    | Default | Description                                         |
| -------------- | ------- | ------- | --------------------------------------------------- |
| `node_version` | string  | `lts/*` | Node.js version                                     |
| `is_build`     | boolean | `true`  | Download the `dist` artifact before releasing       |
| `is_publish`   | boolean | `false` | Exposed as the `PUBLISH` env var to semantic-release |

| Secret                     | Required | Description                                                       |
| -------------------------- | -------- | ----------------------------------------------------------------- |
| `SEMANTIC_BOT_CLIENT_ID`   | yes      | GitHub App Client ID used to push tags and create releases        |
| `SEMANTIC_BOT_PRIVATE_KEY` | yes      | GitHub App private key                                            |
| `NPM_TOKEN`                | no       | npm token, only if the package doesn't use npm trusted publishing |

The GitHub App must be installed on the repository with these **repository permissions**: `Contents: Read and write` (tags, releases), `Issues: Read and write` and `Pull requests: Read and write` (semantic-release comments on released issues and PRs). Token generation fails if the app lacks one of them.

### `pr-checks.yml`

Checks the pull request metadata, separately from the build, so that editing the PR title or description re-runs only these fast checks. Jobs are skipped outside `pull_request` / `pull_request_target` events.

- **PR title**: validated with the project's commitlint config (runs `npm ci --ignore-scripts`)
- **Linked issue**: fails if the PR isn't linked to an issue (`Closes #123` or the "Development" sidebar). Skipped for bot PRs (Renovate, Dependabot…)

| Input                     | Type    | Default | Description                   |
| ------------------------- | ------- | ------- | ----------------------------- |
| `node_version`            | string  | `lts/*` | Node.js version               |
| `is_pr_title_checked`     | boolean | `true`  | Validate the PR title         |
| `is_linked_issue_checked` | boolean | `true`  | Require a linked issue        |

### Example

```yaml
name: CI

on:
  pull_request:
    types: [opened, edited, synchronize, reopened]
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  pr-checks:
    if: github.event_name == 'pull_request'
    uses: vfourny/.github/.github/workflows/pr-checks.yml@v2.0.0
    permissions:
      contents: read
      issues: read
      pull-requests: read

  build:
    if: github.event_name != 'pull_request' || github.event.action != 'edited'
    uses: vfourny/.github/.github/workflows/build.yml@v2.0.0

  release:
    if: github.event_name == 'push'
    needs: build
    uses: vfourny/.github/.github/workflows/release.yml@v2.0.0
    permissions:
      contents: read
      id-token: write
    secrets:
      SEMANTIC_BOT_CLIENT_ID: ${{ secrets.SEMANTIC_BOT_CLIENT_ID }}
      SEMANTIC_BOT_PRIVATE_KEY: ${{ secrets.SEMANTIC_BOT_PRIVATE_KEY }}
```

> Reusable workflows can't get more permissions than the caller grants, so the caller job must grant the permissions listed above.

## Renovate preset

To use the shared configuration in another repository, put this in its `renovate.json`:

```json
{
  "extends": ["github>vfourny/.github"]
}
```

What the preset does:

- Pins GitHub Actions to commit SHAs
- Waits 3 days before proposing a release (30 days for major updates)
- Groups patch/minor updates and automerges them on Monday night (00:00–03:59)
- Never automerges major updates or security fixes; security fixes skip the waiting period
- Updates of these shared workflows skip the waiting period

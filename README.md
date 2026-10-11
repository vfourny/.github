# .github

Shared GitHub configuration for all `vfourny` repositories:

- **Reusable workflows** (build, release, linked-issue check)
- **Default community files** (issue templates, PR template, `CONTRIBUTING.md`, `SECURITY.md`), applied to every repository that doesn't define its own
- **Shared Renovate preset** (`default.json`)

## Reusable workflows

### `build.yml`

Installs dependencies, then checks formatting, lint, PR title, build, types and tests, and uploads `dist` as an artifact.

| Input                 | Type    | Default | Description                                               |
| --------------------- | ------- | ------- | --------------------------------------------------------- |
| `node_version`        | string  | `lts/*` | Node.js version passed to `actions/setup-node`            |
| `is_build`            | boolean | `true`  | Run `npm run build`                                       |
| `is_build_upload`     | boolean | `true`  | Upload `dist` as the `dist` artifact (requires `is_build`) |
| `is_type_checked`     | boolean | `true`  | Run `npm run ts:check`                                    |
| `is_tested`           | boolean | `true`  | Run `npm run test`                                        |
| `is_pr_title_checked` | boolean | `true`  | Validate the PR title with commitlint (PR events only)    |

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
| `SEMANTIC_BOT_APP_ID`      | yes      | GitHub App ID used to push tags and create releases               |
| `SEMANTIC_BOT_PRIVATE_KEY` | yes      | GitHub App private key                                            |
| `NPM_TOKEN`                | no       | npm token, only if the package doesn't use npm trusted publishing |

### `check-linked-issues.yml`

Fails if the pull request isn't linked to an issue (`Closes #123` or the "Development" sidebar). Only runs on `pull_request` / `pull_request_target` events.

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
  linked-issue:
    if: github.event_name == 'pull_request'
    uses: vfourny/.github/.github/workflows/check-linked-issues.yml@main
    permissions:
      issues: read
      pull-requests: read

  build:
    uses: vfourny/.github/.github/workflows/build.yml@main

  release:
    if: github.event_name == 'push'
    needs: build
    uses: vfourny/.github/.github/workflows/release.yml@main
    permissions:
      contents: read
      id-token: write
    secrets:
      SEMANTIC_BOT_APP_ID: ${{ secrets.SEMANTIC_BOT_APP_ID }}
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

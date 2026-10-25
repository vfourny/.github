# .github

Shared GitHub configuration for all `vfourny` repositories:

- **Reusable workflows** (build, release, PR checks, Prisma check)
- **Default community files** (issue templates, PR template, `CONTRIBUTING.md`, `SECURITY.md`), applied to every repository that doesn't define its own
- **Shared Renovate preset** (`default.json`)

## Versioning

This repository is released with its own `release.yml` (release-please, auto-merged): merging a PR on `main` whose Conventional Commits title calls for a release (`fix:` → patch, `feat:` → minor, `!` or `BREAKING CHANGE` → major) opens a release PR, which merges itself once its checks pass and creates the `vX.Y.Z` tag, the GitHub release and the `CHANGELOG.md` entry. Other types (`docs:`, `chore:`, `ci:`, `test:`…) don't release.

**Always reference a release, never `@main`**, so a change here can't break your repositories. With the Renovate preset below, references are pinned to a commit SHA and updated through PRs (patch/minor automerged, majors reviewed manually):

```yaml
uses: vfourny/.github/.github/workflows/build.yml@v2.0.0
```

## Reusable workflows

### `build.yml`

Installs dependencies, checks formatting, lint, unused code (knip) and types, builds, runs the tests, and uploads `dist` as an artifact.

| Input                 | Type    | Default | Description                                               |
| --------------------- | ------- | ------- | --------------------------------------------------------- |
| `node_version`        | string  | `lts/*` | Node.js version passed to `actions/setup-node`            |
| `package_manager`     | string  | `npm`   | `npm` or `pnpm` (pnpm version read from `packageManager`) |
| `is_build`            | boolean | `true`  | Run `npm run build`                                       |
| `is_build_upload`     | boolean | `true`  | Upload `dist` as the `dist` artifact (requires `is_build`) |

Each check runs only if the repository defines its script (`run --if-present`): `format:check`, `lint`, `knip`, `ts:check` and `test`. To skip a check, don't define the script. `build` runs when `is_build` is set. Scripts run with `npm run` or `pnpm run` depending on `package_manager`.

### `release.yml`

Releases with [release-please](https://github.com/googleapis/release-please):

1. Every push on `main` creates or updates a release PR (`chore(main): release X.Y.Z`): version in `package.json` and CHANGELOG, computed from the Conventional Commits since the last release.
2. Merging it creates the `vX.Y.Z` tag and the GitHub release.

Two modes, through `is_auto_merged`:

- `false` (default), **on demand**: you merge the release PR when you want to ship (deployed apps).
- `true`, **automatic**: auto-merge (squash) is enabled on the release PR, which merges itself as soon as its checks pass, so every releasable merge ships (npm packages). Requires **Settings › General › Allow auto-merge**.

The workflow never pushes to `main` (the version commit arrives through the release PR, so branch rulesets apply as usual) and doesn't publish or deploy anything: chain your own job on its outputs.

| Input            | Type    | Default | Description                              |
| ---------------- | ------- | ------- | ---------------------------------------- |
| `is_auto_merged` | boolean | `false` | Enable auto-merge on the release PR      |

| Secret          | Required | Description                                                                 |
| --------------- | -------- | --------------------------------------------------------------------------- |
| `RELEASE_TOKEN` | yes      | Fine-grained PAT: **Contents**, **Pull requests**, **Issues** (read and write) |

| Output            | Description                              |
| ----------------- | ---------------------------------------- |
| `release_created` | `'true'` when this run created a release |
| `tag_name`        | Created tag (e.g. `v1.2.0`)              |
| `sha`             | Released commit SHA                      |

Why a PAT and not `GITHUB_TOKEN`: what `GITHUB_TOKEN` creates doesn't trigger workflows, so the release PR would get no CI (required checks never pass) and, with auto-merge, merging it wouldn't run the release. One PAT can cover all repositories ("Only select repositories"); it expires (one year max), and so do releases with it: renew it and update the `RELEASE_TOKEN` secret of each repository.

Files to add in each repository (release-please reads them from the repository):

`release-please-config.json`

```json
{
  "$schema": "https://raw.githubusercontent.com/googleapis/release-please/main/schemas/config.json",
  "packages": {
    ".": {
      "changelog-path": "docs/CHANGELOG.md",
      "include-component-in-tag": false,
      "release-type": "node"
    }
  }
}
```

`.release-please-manifest.json`: the current version (from `package.json`, or the latest `vX.Y.Z` tag)

```json
{ ".": "0.1.0" }
```

If the repository has no `vX.Y.Z` tag yet, add `"bootstrap-sha": "<HEAD of main>"` to the config, so the first CHANGELOG doesn't list the whole history. Add the CHANGELOG and the manifest to `.prettierignore` if Prettier checks them: release-please writes them in its own format.

Caller, with an npm publication chained on the release:

```yaml
name: CD

on:
  push:
    branches: [main]

permissions:
  contents: read

jobs:
  release:
    uses: vfourny/.github/.github/workflows/release.yml@vX.Y.Z
    with:
      is_auto_merged: true
    secrets:
      RELEASE_TOKEN: ${{ secrets.RELEASE_TOKEN }}

  publish:
    needs: release
    if: ${{ needs.release.outputs.release_created == 'true' }}
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # npm trusted publishing (OIDC)
    steps:
      - uses: actions/checkout@v7
        with:
          persist-credentials: false
      - uses: actions/setup-node@v7
        with:
          node-version: 'lts/*'
          registry-url: 'https://registry.npmjs.org'
      - run: npm ci
      - run: npm publish
```

## Upgrading to v4

v4 replaces semantic-release with release-please in `release.yml`:

- Inputs `node_version`, `is_build`, `is_publish` and secrets `SEMANTIC_BOT_CLIENT_ID` / `SEMANTIC_BOT_PRIVATE_KEY` / `NPM_TOKEN` are gone; the workflow takes `is_auto_merged` and a `RELEASE_TOKEN` PAT.
- Add `release-please-config.json` + `.release-please-manifest.json` (above), drop the semantic-release config and dependencies.
- Publishing (npm) moves to a job of the caller, chained on `release_created`.
- The GitHub App used by v3 is no longer needed.

### `pr-checks.yml`

Checks the pull request metadata, separately from the build, so that editing the PR title or description re-runs only these fast checks. Jobs are skipped outside `pull_request` / `pull_request_target` events.

- **PR title**: validated with the project's commitlint config (dependencies installed with `--ignore-scripts`)
- **Linked issue**: fails if the PR isn't linked to an issue (`Closes #123` or the "Development" sidebar). Skipped for bot PRs (Renovate, Dependabot…)

| Input                     | Type    | Default | Description                   |
| ------------------------- | ------- | ------- | ----------------------------- |
| `node_version`            | string  | `lts/*` | Node.js version               |
| `package_manager`         | string  | `npm`   | `npm` or `pnpm`               |
| `is_pr_title_checked`     | boolean | `true`  | Validate the PR title         |
| `is_linked_issue_checked` | boolean | `true`  | Require a linked issue        |

### `prisma-check.yml`

Fails when the Prisma schema and the migrations folder are out of sync (the schema changed but no migration was created with `prisma migrate dev`). It replays every migration on an empty PostgreSQL service with `prisma migrate deploy`, then runs `prisma migrate diff --from-config-datasource --to-schema <schema_path> --exit-code`. No secret and no shadow database needed.

Requirements: PostgreSQL, Prisma 7+ (`prisma.config.ts` with a `datasource.url` read from `DIRECT_URL` or `DATABASE_URL`: the workflow sets both to the CI database).

| Input              | Type   | Default                | Description                                             |
| ------------------ | ------ | ---------------------- | ------------------------------------------------------- |
| `node_version`     | string | `lts/*`                | Node.js version                                         |
| `package_manager`  | string | `npm`                  | `npm` or `pnpm`                                         |
| `schema_path`      | string | `prisma/schema.prisma` | Schema file or folder (multi-file schema)               |
| `config_path`      | string | `''`                   | `prisma.config.ts` path when not at the root (monorepo) |
| `postgres_version` | string | `17`                   | `postgres` Docker image tag, match the production major |

Run it on every PR, without a `paths` filter, if it's a required status check: a skipped required check stays pending and blocks the merge.

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
    uses: vfourny/.github/.github/workflows/release.yml@v4.0.0
    with:
      is_auto_merged: true
    secrets:
      RELEASE_TOKEN: ${{ secrets.RELEASE_TOKEN }}
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

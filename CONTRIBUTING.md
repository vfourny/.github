# Contributing

Thanks for your interest in contributing!

## Workflow

1. Open an issue first (bug report, user story, task or documentation request) using the provided templates.
2. Create a branch from `main`.
3. Open a pull request **linked to the issue** (`Closes #<issue>` in the description or via the "Development" sidebar). A CI check enforces it.
4. Wait for the CI to pass and for a review from a code owner.

## Commit and PR title conventions

PR titles must follow [Conventional Commits](https://www.conventionalcommits.org/)
(`feat: ...`, `fix: ...`, `chore: ...`). They are validated with commitlint and used by
semantic-release to compute versions and changelogs.

## Before pushing

Run the checks defined by the repository before pushing. For Node.js projects using the shared `build.yml` workflow, these are:

```sh
npm run format:check
npm run lint
npm run ts:check
npm run test
npm run build
```

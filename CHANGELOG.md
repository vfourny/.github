# Changelog

## [4.1.0](https://github.com/vfourny/.github/compare/v4.0.1...v4.1.0) (2026-10-25)


### Features

* **workflows:** add config_path to prisma-check ([#102](https://github.com/vfourny/.github/issues/102)) ([9f688bc](https://github.com/vfourny/.github/commit/9f688bca1ee5ec462f553132b469420d9083be8b))
* **workflows:** add is_vars_exported to build ([#104](https://github.com/vfourny/.github/issues/104)) ([96b90e9](https://github.com/vfourny/.github/commit/96b90e9655fe4c4722f818769f4615e686894c40))

## [4.0.1](https://github.com/vfourny/.github/compare/v4.0.0...v4.0.1) (2026-10-17)


### Bug Fixes

* **release:** auto merge step ([#98](https://github.com/vfourny/.github/issues/98)) ([785d866](https://github.com/vfourny/.github/commit/785d866de0531824cf86a3228a7b6253a1b67629))

## [4.0.0](https://github.com/vfourny/.github/compare/v3.1.0...v4.0.0) (2026-10-16)


### ⚠ BREAKING CHANGES

* **workflows:** release.yml inputs (node_version, is_build, is_publish) and secrets (SEMANTIC_BOT_*, NPM_TOKEN) are replaced by is_auto_merged and RELEASE_TOKEN; callers add the release-please config files and publish in their own job.

### Features

* **workflows:** release with release-please instead of semantic-release ([#94](https://github.com/vfourny/.github/issues/94)) ([2159b55](https://github.com/vfourny/.github/commit/2159b554de338fe20d95f33876e74d1f93f6f657))

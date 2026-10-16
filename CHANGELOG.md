# Changelog

## [4.0.0](https://github.com/vfourny/.github/compare/v3.1.0...v4.0.0) (2026-10-16)


### ⚠ BREAKING CHANGES

* **workflows:** release.yml inputs (node_version, is_build, is_publish) and secrets (SEMANTIC_BOT_*, NPM_TOKEN) are replaced by is_auto_merged and RELEASE_TOKEN; callers add the release-please config files and publish in their own job.

### Features

* **workflows:** release with release-please instead of semantic-release ([#94](https://github.com/vfourny/.github/issues/94)) ([2159b55](https://github.com/vfourny/.github/commit/2159b554de338fe20d95f33876e74d1f93f6f657))

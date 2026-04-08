# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-04-08

### Added

- **`/gs-nav` skill** — navigate between stack branches via interactive picker or shorthand arguments (`next`, `prev`, `first`, `last`, by number, or `main`). Local-only, no `gh` required. ([#4](https://github.com/tbekaert/claude-ghstack/pull/4))
- **CI wait/polling in `/gs-merge`** — when CI checks are pending, the skill now asks whether to wait and polls every 30 seconds instead of stopping immediately. Includes 10-minute timeout and user interrupt handling. ([#2](https://github.com/tbekaert/claude-ghstack/pull/2))

### Fixed

- Use `gh` state names (`SUCCESS`/`FAILURE`/`PENDING`) in gs-merge CI checks instead of incorrect `pass`/`fail`
- Use `--json name,state` for structured CI check parsing

## [0.1.0] - 2026-04-08

### Added

- Eight slash-command skills for the full stacked PR lifecycle:
  - `/gs-create` — create a new branch stacked on the current one
  - `/gs-insert` — insert a branch at a chosen position in the stack
  - `/gs-move` — reorder or detach a branch in the stack
  - `/gs-submit` — push branches and create/update PRs on GitHub
  - `/gs-sync` — rebase the stack after upstream changes
  - `/gs-merge` — merge approved PRs with retarget, cleanup, and rebase
  - `/gs-log` — display the stack graph with PR status
  - `/gs-help` — show the quick reference card
- Four shared pattern docs: stack discovery, prerequisites, staging, conflict handling
- Plugin marketplace support (`marketplace.json`) for easy installation
- `CLAUDE.md` with project architecture and skill conventions
- Comprehensive README with comparison to other stacked PR tools

[0.2.0]: https://github.com/tbekaert/claude-ghstack/compare/v0.1.0...v0.2.0
[0.1.0]: https://github.com/tbekaert/claude-ghstack/releases/tag/v0.1.0

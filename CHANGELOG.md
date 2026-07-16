# Changelog

All notable changes to `jharness-tools` are recorded here.

The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and
versions follow [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Added explicit `ReadTool`, `GlobTool`, and `GrepTool` presets with workspace
  containment, deterministic bounded results, cooperative cancellation, and stable
  model-visible failures.
- Added strict UTF-8 text handling, shared search exclusions, structured results, and
  parallel-safe execution and filesystem risk declarations.
- Added opened-handle workspace revalidation, bounded file and line output, and
  time-limited regular-expression evaluation.
- Added no-follow directory traversal, iterative and complexity-bounded glob matching,
  and Host-configured search time, entry, and byte budgets.
- Added direct runtime dependencies on `jharness-kernel` contracts and the `regex`
  engine; `jharness-toolkit` remains development-only.
- Added explicit `EditTool` and `WriteTool` presets with required SHA-256 compare-and-set
  inputs, serial mutation facts, conservative approval risk declarations, and bounded
  structured receipts.
- Added per-target mutation coordination, no-follow parent traversal, same-directory
  staged writes, atomic no-clobber creation, optimistic atomic replacement, cooperative
  pre-commit cancellation, and stable mutation failures.
- Added raw-byte SHA-256 results to both the structured and model-visible `Read` output,
  allowing models to pass explicit state into `Edit` and conditional `Write` without
  hidden Host memory.
- Added the serial `BashTool` preset with Host-bounded command length, duration, stdout,
  and stderr; structured exit observations; stable timeout, cancellation, spawn, and
  working-directory failures; and conservative subprocess, network, filesystem, and
  approval risk facts.
- Added non-interactive Bash process execution with independent pipe draining and
  truncation accounting, Host-controlled environment selection, workspace-rooted
  initial directories, bounded cleanup waits, and process-group or Windows Job/fallback
  termination for managed descendants.
- Added the durable `AskQuestionTool` preset with Host-selectable confirm, choice, text,
  number, date, scale, and ranking interactions; stable request identifiers; strict
  semantic validation; and no filesystem, network, subprocess, or approval effects.
- Added immutable question request/response values plus validated answer rendering and
  exact checkpoint-resume helpers, including explicit answered and cancelled outcomes.
- Added the Host-mediated `Agent`, `AgentGet`, `AgentWait`, and `AgentCancel` presets
  with strict bounded contracts, foreground durable waits, background acceptance,
  non-blocking snapshots, acknowledged cancellation, and stable task references.
- Added immutable Agent request/snapshot values, the narrow `AgentBackend` execution
  port, and strict checkpoint extraction plus canonical parent-resume helpers. Child
  Runtime inheritance, storage, supervision, authorization, and cancellation remain
  explicitly Host-owned.

### Fixed

- Corrected Darwin opened-path validation to use the 1,024-byte buffer accepted by
  CPython's `fcntl` interface and the platform `F_GETPATH` contract.

## [0.1.0] - 2026-07-16

### Added

- Created the independent `jharness-tools` project and `jharness.tools` package
  scaffold without preset implementations.

[Unreleased]: https://github.com/Ezio2000/jharness-tools/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/Ezio2000/jharness-tools/releases/tag/v0.1.0

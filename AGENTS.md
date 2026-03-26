# AGENTS.md

This file defines repository-specific guidance for autonomous coding agents working in this repository.

## Project overview

- Repository: Betaflight firmware (embedded C/C++).
- Primary build system: GNU `make` at repository root.
- Main firmware sources: `src/main`.
- Tests: `src/test` (invoked via root Makefile).

## Branch and PR expectations

- Work on a feature branch, not `master`.
- Open pull requests against `master`.
- Keep changes focused and small.
- Follow coding style guidance: https://www.betaflight.com/docs/development/CodingStyle

## Build and test commands

Run commands from the repository root unless noted otherwise.

### Discover targets and help

- `make help`
- `make targets`

### Build firmware

- Build one target (default firmware output): `make TARGET=<TARGET_NAME>`
- Build explicit hex output: `make hex TARGET=<TARGET_NAME>`
- Clean one target: `make TARGET=<TARGET_NAME> clean`
- Full clean: `make clean`

Example:

- `make TARGET=SPEEDYBEEF405WING`

### Run tests

- Full test suite: `make test`
- Representative subset: `make test-representative`
- Specific test target: `make test_<name>`
- Test help/listing: `make test_help`

### Static analysis

- Run cppcheck: `make cppcheck`

## Code-change guidance

- Prefer minimal, targeted diffs.
- Avoid broad refactors unless required by the task.
- Do not modify vendored third-party code in `lib/main` unless the task explicitly requires it.
- Keep warnings clean; this repo treats warnings as errors in normal build paths.

## Validation expectations for agents

- For code changes, run at least one high-signal command that executes the modified path.
- If changing core logic, prefer:
  - one target firmware build relevant to the changed platform or subsystem, and
  - relevant test suite command(s) from `make test_*` or `make test-representative`.
- For docs-only changes, lightweight validation (file checks/spell/format sanity) is sufficient.

## Cursor Cloud specific instructions

- Use fast repo-aware search tools (`rg`, glob) instead of recursive shell scans.
- Do not run the entire target matrix unless explicitly requested; builds are expensive.
- Prefer narrow, high-signal checks:
  - `make TARGET=<TARGET_NAME>` for firmware-impacting changes.
  - `make test-representative` or targeted `make test_<name>` for test-impacting changes.
- If toolchain/dependency issues block local validation, report exact failing command and stderr, and continue with the highest-signal checks that are still runnable.
- If making UI changes in web assets or docs with rendered output, include a screenshot or video artifact; otherwise terminal evidence is sufficient.


# AGENTS.md

Guidance for agentic coding assistants in `badness-action`.

## Scope

- Repo type: composite GitHub Action.
- Purpose: install `badness` and run format/lint checks on LaTeX sources.
- Primary files: `action.yml`, `scripts/install-badness.sh`, `scripts/install-badness.ps1`.
- CI coverage: Linux/macOS/Windows on x64 and ARM64.

## Repository Map

- `action.yml`: action API (inputs/outputs) and execution steps.
- `scripts/install-badness.sh`: Unix installer (verifies checksum + provenance).
- `scripts/install-badness.ps1`: Windows installer (verifies checksum + provenance).
- `.github/workflows/ci.yml`: lint, integration tests, and the versionary release job.
- `.github/workflows/update-major-minor-tags.yml`: release tag maintenance.
- `fixtures/ok.tex`, `fixtures/bad.tex`: expected pass/fail fixtures. `ok.tex`
  passes both `badness format --check` and `badness lint`; `bad.tex` fails
  both. Fixtures are pinned to LF via `.gitattributes` (badness emits LF).
- `versionary.jsonc`: versionary release config (`simple` strategy).
- `version.txt`: the current version, managed by versionary.

## Tooling Assumptions

- No `package.json`, `Makefile`, or Python project files.
- No compile/build artifact pipeline; tests are workflow-driven.
- Installer smoke checks require network access.

## Lint and Validation

Run from repo root.

The `lint` job in `ci.yml` runs all of these; run them locally before pushing.

- Shell syntax: `sh -n scripts/install-badness.sh`
- ShellCheck: `shellcheck scripts/install-badness.sh`
- PowerShell parse check:
  `pwsh -NoLogo -NoProfile -Command "[void][ScriptBlock]::Create((Get-Content -Raw 'scripts/install-badness.ps1'))"`
- Workflow lint: `actionlint`

## Test

- Main workflow: `.github/workflows/ci.yml`.
  - `test-pass` should succeed with `fixtures/ok.tex`.
  - `test-fail` should fail with `fixtures/bad.tex` (failure is asserted).
- Focused Unix smoke check without CI:
  - `tmpdir="$(mktemp -d)" && BADNESS_INSTALL_DIR="$tmpdir" BADNESS_VERIFY_CHECKSUM=false bash scripts/install-badness.sh && "$tmpdir/badness" --version`

## Code Style Guidelines

- Preserve Unix/Windows behavior parity; keep OS conditionals explicit.
- YAML: 2-space indent, kebab-case input names, string booleans (`"true"`/`"false"`).
- Shell: POSIX `sh`, prologue `#!/usr/bin/env sh` + `set -eu`, `case` for OS/arch
  branching, quote expansions, HTTPS-only downloads, `trap` cleanup.
- PowerShell: `$ErrorActionPreference = 'Stop'`, camelCase names, explicit
  cmdlets, `try/finally` cleanup, throw on unsupported architecture.
- Env vars: `BADNESS_*` (UPPER_SNAKE_CASE).
- Update `README.md` when behavior or the input/output API changes.
- Use Conventional Commits (`feat:`, `fix:`, `chore:`).

## Security

- Download artifacts only over HTTPS from GitHub Releases.
- Verify downloads against the published `.sha256` sidecar; a mismatch aborts,
  a missing sidecar (older releases) warns and continues.
- Verify build provenance with `gh attestation verify` when `gh` is available;
  a failing attestation aborts, a missing one or missing `gh` warns.
- Never log secrets/tokens.
- Treat release/tag automation edits as high risk.

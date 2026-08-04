# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is the **`.github` repository** of the `request-dl` GitHub organization — a special-purpose repo that GitHub uses to provide organization-wide defaults and community health files. It contains **no application/package source code**. The actual RequestDL library lives in sibling repositories:

- [`request-dl/request-dl-nio`](https://www.github.com/request-dl/request-dl-nio) — the RequestDL Swift package (SwiftNIO-based declarative networking layer, inspired by SwiftUI's declarative style)
- [`request-dl/swift-openapi-request-dl-nio`](https://www.github.com/request-dl/swift-openapi-request-dl-nio) — OpenAPI transport for RequestDL

There is nothing to build, lint, or test in this repository itself — its only "product" is the reusable CI workflow and org-wide config files below.

## Repository contents

- `README.md` / `profile/README.md` — identical org-profile READMEs (the profile one is what renders on the org's GitHub page).
- `CONTRIBUTING.md` — PR contribution guidelines used across org repos.
- `CODE_OF_CONDUCT.md`, `LICENSE`, `.github/FUNDING.yml` — standard community health files.
- `.github/dependabot.yaml` — weekly Dependabot updates for `github-actions` and `swift` package ecosystems.
- `.github/PULL_REQUEST_TEMPLATE.md` — default PR template applied across the org.
- `.github/workflows/swift-ci.yaml` — **the main asset of this repo**: a reusable GitHub Actions workflow (`workflow_call`) that other RequestDL repositories call to run their CI.

## `swift-ci.yaml`: the reusable CI workflow

This workflow is invoked with `uses: request-dl/.github/.github/workflows/swift-ci.yaml@<ref>` from other repos, driven entirely by `inputs`/`secrets` — there's no hardcoded package name or scheme. Changes here affect CI for every repo that calls it, so treat edits as changes to shared infrastructure.

Job pipeline and dependency order:

```
format-check
     │
     ▼
breaking-changes
     │
     ├──► apple-tests (matrix: iOS/iPadOS/watchOS/tvOS/visionOS/macOS/macOS-Catalyst)
     ├──► third-party-tests (matrix: Linux, Windows commented out)
     └──► linkage-test (Linux only)
               │
               ▼
         coverage-upload
```

- **`format-check`** — runs `swift format lint --strict --recursive` using `.swift-format` config from the calling repo (falls back to defaults if absent). Gate for everything downstream.
- **`breaking-changes`** — runs `swift package diagnose-api-breaking-changes <latest_tag>` against the last git tag. Auto-skipped on: push to main (not a PR), major-version tag bumps (when `skip-on-major-update` is true), PRs labeled `major-release`/`breaking-change`/`release/major`, or branches named `release/v*`, `v*`, `major`, `next-major`. Needs `format-check` to succeed or be skipped.
- **`apple-tests`** — matrix job across all Apple platforms via `xcodebuild build-for-testing` / `test-without-building`, using `xcbeautify` for readable output. When `apple-coverage-enabled`, extracts per-line coverage from the generated `.xcresult` via `xcrun xccov view --archive` (as a `coverage.xccov` text file) and uploads it as an artifact — since Xcode 26, coverage data no longer sits as a loose `.profdata` file under derived data, so `xccov` reads it straight from the `.xcresult` bundle instead of relying on `find`-ing a `.profdata` on disk.
- **`third-party-tests`** — runs `swift test --enable-code-coverage` (currently only Linux; Windows matrix entry exists but is commented out). Coverage artifacts here are still raw `.profdata` + `.xctest` bundles copied from `.build`, since SwiftPM's own toolchain (not `xcodebuild`/`.xcresult`) produces them.
- **`linkage-test`** — Linux-only check that builds a throwaway executable package depending on the target package and asserts (via `ldd`) that the resulting binary does **not** link `libFoundation.so`. This enforces that the package uses `FoundationEssentials` rather than full `Foundation` on Linux (which would otherwise require importing `FoundationNetworking` for `URLSession` APIs).
- **`coverage-upload`** — downloads all coverage artifacts and converts each to `.lcov`, then uploads to Codecov. Per-artifact-dir: if a `coverage.xccov` file is present (from `apple-tests`), a small embedded Python script (`xccov_to_lcov.py`) parses its `xccov view --archive` text output into `SF:`/`DA:`/`end_of_record` LCOV records; otherwise (from `third-party-tests`) it falls back to the classic `.profdata` + `.xctest` + `xcrun llvm-cov export` path.

All downstream jobs use `if: always() && ...` guards combined with `needs.<job>.result == 'success' || 'skipped'` so that a skipped upstream step (e.g. `breaking-changes` skipped on a major bump) doesn't block later jobs — only an actual **failure** does.

### Key `workflow_call` inputs

| Input | Purpose |
|---|---|
| `name` / `product-name` | Package/repo identifier vs. importable library name (used to scaffold the linkage-test package) |
| `swift-version` / `format-swift-version` | Swift toolchain versions (format check can use a newer/looser version than the build) |
| `enable-format` / `enable-breaking-changes` / `enable-apple-tests` / `enable-third-party-tests` / `enable-linkage-test` / `enable-coverage-upload` | Per-repo opt-in/out of each pipeline stage |
| `force-skip-breaking-changes` | Manual override to bypass the breaking-changes check |
| `apple-coverage-enabled` / `third-party-coverage-enabled` | Per-platform coverage artifact collection toggles |

When editing this workflow, keep the `needs:`/`if:` guard pattern consistent across jobs — the `always()` + result-check idiom is what lets the pipeline degrade gracefully when a stage is disabled or skipped rather than being treated as a hard failure.

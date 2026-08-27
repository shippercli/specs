# Spec: GitHub Actions Integration

**Issue:** MAR-40
**Date:** 2026-04-24
**Status:** In Review

## Overview

Shipper ships two GitHub Actions integration points: (1) reusable workflow files in `.github/workflows/` for CI/CD pipelines, and (2) a composite GitHub Action at `.github/actions/shipper/` that allows any repository to invoke Shipper without PHP/Composer setup.

## Current Implementation

### Workflow Files

| Workflow | Trigger | Purpose |
|---|---|---|
| `build-release.yml` | Tag push (`v*`) | Builds PHAR binary, creates GitHub release with binary attached |
| `ci.yml` | Push/PR | Runs tests and linting |
| `deploy-production.yml` | Push to `main` | Deploys all projects to production via `./shipper apply` |
| `deploy-staging.yml` | Push to `develop` | Deploys all projects to staging |
| `deploy-preview.yml` | PR to `main`/`develop` | Creates PR preview environment |
| `cleanup-preview.yml` | PR closed (`main`/`develop`) | Destroys preview environment on PR close |
| `weekly-cleanup.yml` | Scheduled | Weekly orphan site cleanup |
| `deploy-production-action-example.yml` | Push to `main` | Example using the reusable Shipper Action |

### Reusable Action — `.github/actions/shipper/`

**action.yml** inputs:

| Input | Required | Default | Description |
|---|---|---|---|
| `command` | Yes | — | `validate`, `plan`, `apply`, `destroy` |
| `project` | No | — | Project name from `shipper.yml` |
| `profile` | No | — | Profile name (production, staging, preview) |
| `force` | No | `false` | Skip confirmation prompts |
| `version` | No | `latest` | CLI version (tag or `latest`) |
| `working-directory` | No | `.` | Directory containing `shipper.yml` |

**Action behavior:**
1. Sets up PHP and Composer for the action
2. Installs `shippercli/cli` and the requested provider packages into an isolated tool directory
3. Verifies and runs that installation's `vendor/bin/shipper` with arguments built safely via bash arrays
4. Outputs `exit-code` for downstream steps

**Referenced as:**
```yaml
uses: shippercli/actions/.github/actions/shipper@v1.0.0   # specific tag
uses: shippercli/actions/.github/actions/shipper@main      # latest dev
uses: shippercli/actions/.github/actions/shipper@939e086   # specific commit
```

## Functional Requirements

**FR-001 — Composer Installation and Verification**
The action installs Shipper and requested provider packages into an isolated Composer tool directory, verifies `vendor/bin/shipper`, and runs `--version` before executing the user's command.

**FR-002 — Safe Argument Handling**
Command arguments are built using a bash array (`ARGS=("$COMMAND")`) to prevent shell injection. No `eval` or string concatenation.

**FR-003 — Exit Code Propagation**
The action writes `exit-code=$EXIT_CODE` to `$GITHUB_OUTPUT` so downstream steps can inspect the result.

**FR-004 — GitHub Release Triggered Build**
`build-release.yml` fires on any tag matching `v*`. It compiles the PHAR and attaches it to a GitHub release via `softprops/action-gh-release@v1`.

**FR-005 — Preview Deployment PR Commenting**
`deploy-preview.yml` uses `actions/github-script@v7` to comment on the PR with the preview URL after deployment.

**FR-006 — Preview Cleanup on PR Close**
`cleanup-preview.yml` triggers on `pull_request: types: [closed]`, destroying the preview site and commenting on the PR.

## Configuration Interface

### Environment Variables

| Variable | Required | Used by |
|---|---|---|
| `PLOI_API_KEY` | Yes | `apply`, `plan`, `destroy` |
| `GITHUB_PR_NUMBER` | Preview only | `apply`/`destroy` for preview profile |
| `GITHUB_HEAD_REF` | Preview only | `apply`/`destroy` for preview profile |

### Full Workflow Example (manual setup vs action)

**Traditional (manual PHP setup):**
```yaml
- uses: actions/checkout@v4
- uses: shivammathur/setup-php@v2
  with:
    php-version: '8.3'
    extensions: mbstring, xml, ctype, json, yaml
    coverage: none
- run: composer install --no-dev
- run: ./shipper apply api --profile=production --force
  env:
    PLOI_API_KEY: ${{ secrets.PLOI_API_KEY }}
```

**Using the Shipper Action:**
```yaml
- uses: actions/checkout@v4
- uses: shippercli/actions/.github/actions/shipper@main
  with:
    command: apply
    project: api
    profile: production
    force: true
    providers: |
      shippercli/provider-ploi:^1.0
  env:
    PLOI_API_KEY: ${{ secrets.PLOI_API_KEY }}
```

### Matrix Strategy

All workflow files use `strategy.matrix.project: [api, frontend]` to deploy multiple projects in parallel, with `needs:` dependencies for sequential ordering (e.g., API before frontend).

## Edge Cases

- **Composer installation fails:** Action exits with error if Shipper or a requested provider cannot be installed
- **Invalid command:** Shipper itself returns exit code 1; action propagates it
- **Version constraint not found:** Composer fails to resolve the CLI or provider constraint and the action exits with error
- **Rate limiting:** GitHub API rate limits are handled by the CLI commands, not by the action
- **Cleanup on force-merge:** When a PR is squash-merged, GitHub fires the `closed` event, triggering cleanup — correct behavior
- **Re-opening PR:** `cleanup-preview.yml` fires on close; reopening creates a new preview via `deploy-preview.yml` (triggered on `opened` and `synchronize`)

## Acceptance Criteria

- [ ] `deploy-preview.yml` fires on PR open and synchronize
- [ ] `cleanup-preview.yml` fires on PR close
- [ ] Action downloads binary from correct URL per version input
- [ ] Action verifies binary with `--version` before executing
- [ ] Action uses bash array for argument construction (no shell injection vectors)
- [ ] Action outputs `exit-code` that downstream steps can inspect
- [ ] `build-release.yml` triggers on any `v*` tag and attaches binary to release
- [ ] `ci.yml` runs on every push and pull request
- [ ] All workflows use `force: true` or `--force` to skip confirmation in CI
- [ ] Preview workflows pass `GITHUB_PR_NUMBER` and `GITHUB_HEAD_REF` env vars

## Open Questions / Potential Concerns

- **Binary integrity:** The action does not verify SHA256 checksum of the downloaded binary. Should checksums be published alongside releases?
- **Action versioning:** The action in the Shipper repo itself (`shippercli/actions/.github/actions/shipper`) is versioned by the same tag as the binary. Using `@main` picks up both the latest action and the latest binary — is this the intended co-versioning story?
- **Concurrent matrix runs:** If two PRs are opened simultaneously, both `deploy-preview.yml` and `cleanup-preview.yml` runs could conflict if they target the same preview domain (shouldn't happen with unique PR numbers, but worth confirming the domain locking is atomic on the provider side)
- **Self-hosting the action:** Users who want to pin to a specific version reference `shippercli/actions/.github/actions/shipper@vX.Y.Z` — this requires the Shipper repo to remain public. Is private hosting intended?

---
description: "Composer-isolated GitHub Action installation, provider discovery, inputs, caching, and workflow requirements."
---

# Spec: GitHub Actions Integration

**Issue:** MAR-40
**Date:** 2026-04-24
**Status:** In Review

## Overview

Shipper provides workflow examples in the CLI repository and a composite action in `shippercli/actions/.github/actions/shipper`. The action sets up its own PHP and Composer runtime, installs the CLI and requested provider packages together in an isolated tool directory, and runs Shipper against the checked-out application.

The application does not need Shipper in its `composer.json`, `composer.lock`, or `vendor` directory.

## Composite Action Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `command` | Yes | - | `validate`, `plan`, `apply`, `status`, `logs`, `rollback`, or `destroy` |
| `project` | No | - | Project name from `shipper.yml` |
| `profile` | No | - | Profile name such as production, staging, or preview |
| `force` | No | `false` | Skip confirmation prompts |
| `release` | No | - | Provider release identifier for rollback |
| `lines` | No | - | Maximum log lines |
| `working-directory` | No | `.` | Directory containing `shipper.yml` |
| `php-version` | No | `8.3` | PHP used for the isolated Shipper installation |
| `cli-version` | No | `^1.0` | Composer constraint for `shippercli/cli` |
| `providers` | Yes | - | Provider packages, one per line with an optional constraint |

## Action Behavior

1. Sets up PHP and Composer independently of the application.
2. Canonicalizes and validates the provider package list.
3. Installs `shippercli/cli` and all providers in one isolated Composer tool directory.
4. Caches that tool installation by OS, PHP version, CLI constraint, and provider list.
5. Confirms requested packages are visible as `shipper-plugin` packages through Composer's `InstalledVersions`.
6. Runs `vendor/bin/shipper --version`, allowlists the requested command, and passes inputs through environment variables and a bash argument array.
7. Runs from `working-directory` without modifying the application's Composer files.

## Reference

Pin consumers to a release tag or commit SHA, not `main`:

```yaml
- uses: actions/checkout@v4
- uses: shippercli/actions/.github/actions/shipper@c2c276e12f831ba2c3377a063d579fede5cc5ecc
  with:
    command: apply
    project: api
    profile: production
    force: true
    cli-version: '^1.0'
    providers: |
      shippercli/provider-cpanel:^1.0
  env:
    CPANEL_API_TOKEN: ${{ secrets.CPANEL_API_TOKEN }}
```

Multiple published providers share one install. The package names below are
illustrative:

```yaml
providers: |
  vendor/provider-one:^1.0
  vendor/provider-two:^1.0
```

## Functional Requirements

**FR-001 - Isolated Composer installation**
The action installs the CLI and providers outside the application checkout and runs that installation's `vendor/bin/shipper`.

**FR-002 - Provider discovery**
Every requested package must be installed with Composer type `shipper-plugin` and visible to `Composer\\InstalledVersions` in the CLI process.

**FR-003 - Safe argument handling**
Inputs are passed through `env:`; commands are allowlisted and arguments are built as a bash array. The action does not evaluate free-form shell input.

**FR-004 - Cache isolation**
The cache key represents the action toolchain only and never hashes the application's lockfile.

**FR-005 - Exit code propagation**
The action writes `exit-code` to `GITHUB_OUTPUT` and returns the CLI exit code.

## Edge Cases

- Empty provider lists fail before installation.
- Invalid package names and non-plugin packages fail with a clear error.
- Conflicting or unavailable Composer constraints fail the single Composer install.
- Unsupported commands fail before the CLI is invoked.
- A missing CLI binary or failed `--version` check fails the action.
- Duplicate provider lines are deduplicated before installation and cache hashing.

## Acceptance Criteria

- [ ] CLI and providers are installed together outside the application checkout.
- [ ] The action runs `$SHIPPER_HOME/vendor/bin/shipper`, never a downloaded PHAR.
- [ ] Application Composer files and `vendor` are not modified.
- [ ] Requested providers are confirmed through `InstalledVersions::getInstalledPackagesByType('shipper-plugin')`.
- [ ] Inputs use environment variables and argv; `command` is allowlisted.
- [ ] Cache identity includes OS, PHP, CLI constraint, and canonical providers.
- [ ] Examples require `providers` and pin `shippercli/actions` to a tag or SHA.
- [ ] Preview workflows pass `GITHUB_PR_NUMBER` and `GITHUB_HEAD_REF`.

## Open Questions

- Should the action publish immutable major tags such as `v1` automatically when a `v1.x.y` release is created?
- Should reusable workflow examples be retained after consumers have migrated to the composite action?
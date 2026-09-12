---
description: "**Issue:** MAR-19 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: CLI Commands

**Issue:** MAR-19
**Date:** 2026-04-23
**Status:** In Review

## Overview

Shipper exposes a declarative deployment workflow through a set of Laravel Zero console commands. These commands form the user-facing interface for the CLI tool, handling configuration validation, deployment planning, deployment execution, site destruction, and orphaned preview site cleanup.

## Current Implementation

### Key Classes / Files

| File | Class | Responsibility |
|------|-------|----------------|
| `app/Commands/ValidateCommand.php` | `ValidateCommand` | Validates `shipper.yml` configuration for all projects and profiles |
| `app/Commands/PlanCommand.php` | `PlanCommand` | Produces a human-readable deployment plan (dry-run) |
| `app/Commands/ApplyCommand.php` | `ApplyCommand` | Validates, shows plan, confirms, then executes a deployment |
| `app/Commands/DeployCommand.php` | `DeployCommand` | Stub command (no-op, placeholder) |
| `app/Commands/DestroyCommand.php` | `DestroyCommand` | Plans and executes site destruction with confirmation |
| `app/Commands/CleanupOrphanedCommand.php` | `CleanupOrphanedCommand` | Removes preview sites whose PRs are closed |
| `app/Commands/Concerns/FormatsDeploymentPlan.php` | `FormatsDeploymentPlan` | Trait: safely renders plan values in output |
| `app/Kernel.php` | `Kernel` | Registers all commands via `commands()` method |

## Functional Requirements

**FR-001 — Configuration Validation**
**As** a developer, **I want** to validate my `shipper.yml` **so that** I can catch misconfiguration before running a deployment.
- Acceptance: Running `shipper validate` returns exit code 0 for a valid config and exit code 1 with descriptive error output for invalid configs.

**FR-002 — Deployment Planning**
**As** a developer, **I want** to preview a deployment plan **so that** I understand what actions will be taken before committing to them.
- Acceptance: Running `shipper plan <project>` outputs provider, project, profile, branch, path, server, domain, and a list of planned actions. Exit code 0 on success.

**FR-003 — Deployment Execution**
**As** a developer, **I want** to deploy a project **so that** my application is provisioned on the target infrastructure.
- Acceptance: Running `shipper apply <project>` validates config, displays plan, prompts for confirmation (unless `--force` is passed), executes deployment, and returns logs on success or descriptive error on failure. Exit code 0 on success.

**FR-004 — Site Destruction**
**As** a developer, **I want** to destroy a deployed site **so that** I can cleanly remove infrastructure.
- Acceptance: Running `shipper destroy <project>` validates config, shows destruction plan, prompts for confirmation (unless `--force`), executes, and returns success/failure. Exit code 0 on success.

**FR-005 — Orphaned Preview Site Cleanup**
**As** a developer, **I want** to automatically clean up preview sites whose PRs are closed **so that** I don't leave dangling infrastructure.
- Acceptance: Running `shipper cleanup-orphaned` requires `GITHUB_TOKEN` and `GITHUB_REPOSITORY` env vars, performs a dry-run to list orphaned sites, and with `--force` or confirmed prompt deletes them. `--dry-run` flag prevents actual deletion.

## Command Interface

### `validate`

```bash
shipper validate [--config=<path>]
```

| Argument/Option | Type | Default | Description |
|-----------------|------|---------|-------------|
| `--config` | option | `shipper.yml` | Path to configuration file |

**Exit codes:** 0 (valid), 1 (invalid or file not found)
**Output:** Project-by-project and profile-by-profile error listing

---

### `plan`

```bash
shipper plan <project> [--profile=<name>] [--config=<path>]
```

| Argument/Option | Type | Default | Description |
|-----------------|------|---------|-------------|
| `<project>` | argument | — | Project name |
| `--profile` | option | `production` | Profile to use |
| `--config` | option | `shipper.yml` | Path to configuration file |

**Exit codes:** 0 (success), 1 (project or profile not found, or validation error)
**Output:** Formatted deployment plan including provider, project, profile, branch, path, server, domain, repository, web directory, root, and action list.

---

### `apply`

```bash
shipper apply <project> [--profile=<name>] [--config=<path>] [--force]
```

| Argument/Option | Type | Default | Description |
|-----------------|------|---------|-------------|
| `<project>` | argument | — | Project name |
| `--profile` | option | `production` | Profile to use |
| `--config` | option | `shipper.yml` | Path to configuration file |
| `--force` | flag | false | Skip confirmation prompt |

**Exit codes:** 0 (success), 1 (failure)
**Behavior:**
1. Validate configuration
2. Display deployment plan
3. If not `--force`, prompt for confirmation
4. Execute deployment
5. Output deployment logs on success or error details on failure

---

### `destroy`

```bash
shipper destroy <project> [--profile=<name>] [--config=<path>] [--force]
```

| Argument/Option | Type | Default | Description |
|-----------------|------|---------|-------------|
| `<project>` | argument | — | Project name |
| `--profile` | option | `production` | Profile to use |
| `--config` | option | `shipper.yml` | Path to configuration file |
| `--force` | flag | false | Skip confirmation prompt |

**Exit codes:** 0 (success), 1 (failure)

---

### `cleanup-orphaned`

```bash
shipper cleanup-orphaned [--config=<path>] [--dry-run] [--force]
```

| Argument/Option | Type | Default | Description |
|-----------------|------|---------|-------------|
| `--config` | option | `shipper.yml` | Path to configuration file |
| `--dry-run` | flag | false | List orphaned sites without deleting |
| `--force` | flag | false | Skip confirmation prompt |

**Environment variables required:** `GITHUB_TOKEN`, `GITHUB_REPOSITORY` (format: `owner/repo`)
**Exit codes:** 0 (success or no orphans found), 1 (failure)

---

### `deploy`

```bash
shipper deploy
```

**Status:** Stub / no-op. Exists as placeholder. Exit code 0.

## Configuration Interface

```yaml
# shipper.yml
providers:
  ploi:
    api_key: "${PLOI_API_KEY}"

projects:
  api:
    provider: ploi
    profiles:
      production:
        server_id: "12345"
        domain: api.example.com
        branch: main
        path: ./examples/api
        repository: github:test/repo
        web_directory: /public
        project_root: /
        deploy_script: deploy.sh
        ssl: true
```

## Data Contracts

### `ValidateCommand::handle(): int`

Returns `self::SUCCESS` (0) or `self::FAILURE` (1).

### `PlanCommand::handle(): int`

Returns `self::SUCCESS` (0) or `self::FAILURE` (1).

### `ApplyCommand::handle(): int`

```php
public function handle(): int
// Uses ApplyDeploymentFlow::handle() for validation + plan
// Uses ApplyDeploymentFlow::execute() for actual deployment
```

### `DestroyCommand::handle(): int`

```php
public function handle(): int
// Uses DestroyDeploymentFlow::handle() for validation + plan
// Uses DestroyDeploymentFlow::execute() for actual destruction
```

### `CleanupOrphanedCommand::handle(): int`

```php
public function handle(): int
// Uses CleanupOrphanedSitesFlow::handle()
// GITHUB_TOKEN and GITHUB_REPOSITORY env vars required
```

### `FormatsDeploymentPlan` trait

```php
private function getPlanValue(array $plan, string $key, string $default = 'unknown'): string
```

## Edge Cases

- **Nonexistent project in `plan`/`apply`/`destroy`:** Returns exit code 1 with message `Project not found: <name>`
- **Nonexistent profile:** Returns exit code 1 with message `Profile not found: <name>`
- **Missing config file:** `ValidateCommand` returns exit 1. Other commands throw `RuntimeException`.
- **Missing `GITHUB_TOKEN` or `GITHUB_REPOSITORY` env vars:** `CleanupOrphanedCommand` returns exit 1 with descriptive error.
- **Confirmation declined:** `apply` and `destroy` output warning and return exit 0 (user cancelled is not an error).
- **Provider validation errors:** Displayed as `Configuration validation failed:` with itemized error list, exit 1.
- **`--force` bypasses confirmation** but still requires a valid plan to proceed.

## Acceptance Criteria

- [ ] `shipper validate` returns 0 for valid config and 1 with error output for invalid config
- [ ] `shipper plan <project>` outputs formatted deployment plan and returns 0 on success
- [ ] `shipper plan nonexistent` returns exit 1 with `Project not found: nonexistent`
- [ ] `shipper plan <project> --profile=nonexistent` returns exit 1 with `Profile not found: nonexistent`
- [ ] `shipper apply <project> --force` validates, displays plan, and executes without prompting
- [ ] `shipper apply` shows `Deployment completed successfully!` and returns exit 0 on success
- [ ] `shipper apply` shows `Deployment failed!` and returns exit 1 on provider failure
- [ ] `shipper destroy` mirrors the apply flow for destruction
- [ ] `shipper cleanup-orphaned --dry-run` lists orphaned sites without deleting them
- [ ] `shipper cleanup-orphaned` requires `GITHUB_TOKEN` and `GITHUB_REPOSITORY` env vars
- [ ] `shipper deploy` is a no-op stub that exits 0

## Open Questions / Potential Concerns

- **DeployCommand is a stub:** It outputs "Starting deployment..." and "Deployment completed successfully!" with no actual deployment logic. This appears to be a placeholder awaiting implementation.
- **No atomic rollback** on deployment failure — if apply partially completes and then fails, there is no automatic cleanup of what was created.
- **CleanupOrphanedCommand** depends on a `CleanupOrphanedSitesFlow` that was referenced but the full source file was not provided in this review scope. Verify that flow handles GitHub API pagination for large PR counts.
- **Confirmation prompt behavior** is consistent across `apply` and `destroy`, but there is no `--yes` short alias; only `--force`. This is fine but worth confirming against user expectations.
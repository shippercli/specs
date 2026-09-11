---
description: "**Issue:** MAR-22 **Date:** 2026-04-24 **Status:** In Review"
---

# Spec: Database Management

**Issue:** MAR-22
**Date:** 2026-04-24
**Status:** In Review

## Overview
Shipper manages database lifecycle for Laravel applications deployed via Ploi or Forge. It creates databases with secure passwords, supports variable interpolation for dynamic naming, and provides full destroy cleanup.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Config/DatabaseConfig.php` | Value object holding name, user, type |
| `app/Deployment/Contracts/DatabaseManagerInterface.php` | Interface for provider implementations |
| `app/Deployment/Concerns/InterpolatesNames.php` | Trait for `${PROJECT_NAME}`, `${PROFILE}`, env var interpolation |
| `app/Deployment/Providers/Ploi/PloiDatabaseManager.php` | Ploi implementation |
| `app/Deployment/Providers/Forge/ForgeDatabaseManager.php` | Forge implementation |
| `app/tests/Unit/Config/DatabaseConfigTest.php` | Unit tests |

## Functional Requirements

**FR-001 — Database Creation**
On `shipper apply`, for each configured database: check if exists on server, create with secure random password (32 hex chars) if not found, link to site.

**FR-002 — Variable Interpolation**
Database name and user support `${PROJECT_NAME}`, `${PROFILE}`, and arbitrary `${ENV_VAR}` placeholders. Undefined env vars resolve to empty string. Trailing underscores removed, multiple underscores collapsed.

**FR-003 — Destroy Cleanup**
On `shipper destroy`, find and delete databases matching interpolated names.

**FR-004 — Idempotent Apply**
Apply is idempotent: if database already exists, it is left unchanged and operation succeeds.

## Configuration Interface

```yaml
databases:
  main:
    name: "myapp_${PROJECT_NAME}_${PROFILE}"
    user: "myapp_${PROJECT_NAME}_${PROFILE}"
    type: mysql  # currently only mysql supported
```

## Data Contracts

```php
// app/Config/DatabaseConfig.php
final class DatabaseConfig
{
    public function __construct(
        private readonly string $name,
        private readonly string $user,
        private readonly string $type = 'mysql',
    ) {}
    public function name(): string;
    public function user(): string;
    public function type(): string;
}

// app/Deployment/Contracts/DatabaseManagerInterface.php
interface DatabaseManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;
    public function apply(SiteContext $site, DeploymentContext $context): OperationResult;
    public function destroy(DeploymentContext $context): OperationResult;
}

// InterpolatesNames trait
private function interpolateDatabaseName(string $name, string $projectName, string $profileName): string
```

## Edge Cases

- **Missing env var:** `${UNDEFINED_VAR}` in name → empty string, then `_` cleaned up
- **Duplicate creation attempt:** `apply()` checks existing, skips if found, returns `OperationResult::ok()`
- **Destroy with missing database:** silently continues, no error returned
- **Empty database list:** `plan()` returns empty array, `apply()` is no-op

## Acceptance Criteria

- [ ] `DatabaseConfig` accepts `name`, `user`, optional `type` (defaults to `mysql`)
- [ ] `DatabaseManagerInterface` implemented by Ploi and Forge providers
- [ ] Variable interpolation resolves `${PROJECT_NAME}`, `${PROFILE}`, env vars
- [ ] Trailing underscores stripped, multiple underscores collapsed after interpolation
- [ ] `apply()` creates database only if it does not already exist
- [ ] `apply()` generates and stores 32-character hex password
- [ ] `destroy()` removes all matching databases
- [ ] `plan()` returns human-readable action descriptions

## Open Questions / Potential Concerns

- PostgreSQL is not yet supported (only `mysql` type works today)
- No database migration support in apply/destroy (future feature)
- No support for separate read/write database configurations
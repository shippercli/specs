# Spec: PHP Version

**Issue:** MAR-33
**Date:** 2026-04-24
**Status:** In Review

## Overview

PHP version management sets the PHP runtime version for a deployed site. Shipper reads the desired version from `ProjectConfig::phpVersion()` and delegates the change to provider-specific implementations (Ploi, Forge). An empty string means no change is made.

## Current Implementation

### Key Classes / Files

| Class/File | Role |
|---|---|
| `App\Deployment\Contracts\PhpVersionManagerInterface` | Interface defining `plan()` and `apply()` |
| `App\Deployment\Providers\Ploi\PloiPhpVersionManager` | Sets PHP version via Ploi API |
| `App\Deployment\Providers\Forge\ForgePhpVersionManager` | Sets PHP version via Forge API |

## Functional Requirements

**FR-001 — No-Op When Version Unset**
When `ProjectConfig::phpVersion()` returns an empty string, `plan()` returns an empty array and `apply()` returns `OperationResult::ok()` without calling the provider API.

**FR-002 — Plan Output**
When a version is set, `plan()` returns `["Set PHP version to: {$phpVersion}"]`.

**FR-003 — Apply via Provider**
`apply()` calls the provider API to set the PHP version on the site. If an empty string is passed to `apply()`, it behaves as a no-op (matching the FR-001 behavior).

**FR-004 — Single Version String**
The version is a simple string (e.g., `'8.2'`, `'8.3'`), not a semver range or array.

## Configuration Interface

```yaml
# Project level (shipper.php)
php_version: "8.3"

# Or empty to skip
php_version: ""
```

## Data Contracts

```php
// PhpVersionManagerInterface
interface PhpVersionManagerInterface {
    /** @return array<string> */
    public function plan(DeploymentContext $context): array;

    public function apply(SiteContext $site, string $phpVersion): OperationResult;
}

// Both implementations follow this pattern:
public function apply(SiteContext $site, string $phpVersion): OperationResult {
    if ($phpVersion === '') {
        return OperationResult::ok();
    }
    // ... provider API call ...
}
```

## Edge Cases

- **Empty string php_version:** No API call made; returns `OperationResult::ok()`.
- **Provider API throws:** Returns `OperationResult::fail()` with descriptive error.
- **Invalid version string:** Not validated at Shipper level; passed directly to provider API.
- **Version unchanged from current:** Provider API may handle this silently or reject; Shipper forwards the call regardless.

## Acceptance Criteria

- [ ] `plan()` returns an empty array when `php_version` is empty
- [ ] `plan()` returns `["Set PHP version to: {$phpVersion}"]` when version is set
- [ ] `apply('')` makes no API call and returns `OperationResult::ok()`
- [ ] Ploi provider calls `$server->sites($site->siteId)->phpVersion($phpVersion)`
- [ ] Forge provider calls `$forge->changeSitePHPVersion($site->serverId, $site->siteId, $phpVersion)`
- [ ] Provider exceptions result in `OperationResult::fail()`

## Open Questions / Potential Concerns

- Should Shipper validate that the PHP version is available on the target server before attempting to set it?
- Is there a minimum PHP version that Shipper itself requires, separate from the deployed application?
- Should `php_version` support multiple versions for different subdirectories or应用的路径?

---
description: "**Issue:** MAR-33 **Date:** 2026-04-24 **Status:** In Review"
---

# Spec: PHP Version

**Issue:** MAR-33
**Date:** 2026-04-24
**Status:** In Review

## Overview

PHP version management sets the PHP runtime version for a deployed site. Shipper reads the desired version from `ProjectConfig::phpVersion()` and delegates the change to the provider when supported. Ploi implements this operation; Forge API v2 does not expose a supported mutation in the provider contract. An empty string means no change is made.

## Current Implementation

### Key Classes / Files

| Class/File | Role |
|---|---|
| `App\Deployment\Contracts\PhpVersionManagerInterface` | Interface defining `plan()` and `apply()` |
| `shippercli/provider-ploi/src/PloiProvider.php` | Checks server-advertised versions and sets the site version through Ploi |
| `shippercli/provider-forge/src/ForgeProvider.php` | Does not claim PHP-version mutation through Forge API v2 |

## Functional Requirements

**FR-001 — No-Op When Version Unset**
When `ProjectConfig::phpVersion()` returns an empty string, `plan()` returns an empty array and `apply()` returns `OperationResult::ok()` without calling the provider API.

**FR-002 — Plan Output**
When a version is set, `plan()` returns `["Set PHP version to: {$phpVersion}"]`.

**FR-003 — Apply via Supported Provider**
Ploi checks the server-advertised versions and sets the site version only when it differs. Forge rejects or reports this configuration as unsupported rather than invoking a removed SDK v1 operation. If an empty string is passed to `apply()`, it behaves as a no-op (matching FR-001).

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
- **Version unchanged from current:** Ploi skips the update call.
- **Forge API v2:** The provider must report the unsupported mutation clearly.

## Acceptance Criteria

- [ ] `plan()` returns an empty array when `php_version` is empty
- [ ] `plan()` returns `["Set PHP version to: {$phpVersion}"]` when version is set
- [ ] `apply('')` makes no API call and returns `OperationResult::ok()`
- [ ] Ploi provider checks available versions and calls `$server->sites($siteId)->phpVersion($phpVersion)` only when needed
- [ ] Forge provider does not claim or invoke a removed PHP mutation endpoint
- [ ] Provider exceptions result in `OperationResult::fail()`

## Open Questions / Potential Concerns

- Should Shipper validate that the PHP version is available on the target server before attempting to set it?
- Is there a minimum PHP version that Shipper itself requires, separate from the deployed application?
- Should `php_version` support multiple versions for different subdirectories or应用的路径?

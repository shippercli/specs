---
description: "**Issue:** MAR-30 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Domain Aliases

**Issue:** MAR-30
**Date:** 2026-04-23
**Status:** In Review

## Overview

Domain aliases allow a deployment to associate additional domain names with a primary site. The `AliasManagerInterface` handles plan and apply operations, reading alias lists from `$context->profile->aliases()`. Aliases are arrays of strings, not config value objects.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Deployment/Contracts/AliasManagerInterface.php` | Interface defining `plan()` and `apply()` |
| `app/Deployment/Providers/Ploi/PloiAliasManager.php` | Ploi implementation |
| `app/Deployment/Providers/Forge/ForgeAliasManager.php` | Forge implementation |

### AliasManagerInterface

```php
interface AliasManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;

    /**
     * @param array<int, string> $aliases
     */
    public function apply(SiteContext $site, array $aliases): OperationResult;
}
```

## Functional Requirements

**FR-001 — Empty alias list is a no-op**
When `$context->profile->aliases() === []`, both `plan()` and `apply()` return early: plan returns `[]`, apply returns `OperationResult::ok()`.

**FR-002 — Plan generates count-aware summary**
`plan()` returns `['Configure N domain alias(es): ALIAS1, ALIAS2, ...']` where N is the count and the aliases are comma-separated.

**FR-003 — Apply passes alias array directly to provider API**
- Ploi: `$server->sites($siteId)->alias()->create($aliases)` — accepts array of strings
- Forge: `$forge->addSiteAliases($serverId, $siteId, $aliases)`

**FR-004 — Apply failures return OperationResult::fail**
Any exception is caught and returns `OperationResult::fail()` with a generic message: `"Failed to configure domain aliases: {$e->getMessage()}"` (rule name not available since apply receives array).

## Data Contracts

```php
// Profile access
$context->profile->aliases(): array<int, string>

// Interface signatures
plan(DeploymentContext $context): array<string>
apply(SiteContext $site, array<int, string> $aliases): OperationResult
```

## Edge Cases

- **Single alias:** Summary says "1 domain alias: ALIAS" (no plural "aliases")
- **Aliases contain duplicates:** Not validated at Shipper level; provider may reject or deduplicate
- **Invalid domain format:** Not validated at Shipper level

## Acceptance Criteria

- [ ] `plan()` returns `[]` when aliases is empty array
- [ ] `plan()` returns properly pluralized summary with correct count
- [ ] `apply()` passes the alias array directly to provider without transformation
- [ ] Both providers use their respective SDK methods
- [ ] Exceptions are caught and returned as `OperationResult::fail()`
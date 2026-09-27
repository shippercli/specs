---
description: "**Issue:** MAR-29 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Nginx Configuration

**Issue:** MAR-29
**Date:** 2026-04-23
**Status:** In Review

## Overview

Nginx configuration management allows projects to specify raw Nginx config snippets that are deployed to the server. The `NginxConfigManagerInterface` handles plan generation and applying raw config strings via provider SDKs.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Deployment/Contracts/NginxConfigManagerInterface.php` | Interface defining `plan()` and `apply()` |
| `shippercli/provider-ploi/src/PloiProvider.php` | Ploi implementation using the raw full-vhost configuration |
| `shippercli/provider-forge/src/ForgeProvider.php` | Forge API v2 does not claim NGINX mutation |

### NginxConfigManagerInterface

```php
interface NginxConfigManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;

    public function apply(SiteContext $site, string $nginxConfig): OperationResult;
}
```

## Functional Requirements

**FR-001 — Empty config is a no-op**
When `$context->project->nginxConfig() === ''`, both `plan()` and `apply()` return early: `plan()` returns `[]`, `apply()` returns `OperationResult::ok()`.

**FR-002 — Plan generates a single summary action**
`plan()` returns `['Update Nginx configuration']` when nginx config is non-empty.

**FR-003 — Apply deploys raw config string**
`apply()` passes the raw `nginxConfig` string directly to the provider's API without transformation:
- Ploi: `$server->sites($siteId)->nginxConfiguration()->update($nginxConfig)`
- Forge: unsupported by the current API v2 provider contract; validation must fail clearly.

**FR-004 — Apply failures return OperationResult::fail**
Any exception is caught and returns `OperationResult::fail()` with the exception message.

## Configuration Interface

```yaml
nginx: |
  server {
    listen 80;
    server_name example.com;
    root /var/www/html;
  }
```

## Data Contracts

```php
// Interface signatures
plan(DeploymentContext $context): array<string>
apply(SiteContext $site, string $nginxConfig): OperationResult

// Project access pattern
$context->project->nginxConfig(): string  # raw Nginx config or ''
```

## Edge Cases

- **Empty string:** Both plan and apply treat as no-op (FR-001)
- **Invalid Nginx syntax:** Not validated by Shipper; provider API may reject or server may fail to reload
- **Very large config:** No size limit enforced at Shipper level

## Acceptance Criteria

- [ ] `plan()` returns `[]` when nginx config is empty string
- [ ] `plan()` returns `['Update Nginx configuration']` when nginx config is non-empty
- [ ] `apply()` does not transform the config string
- [ ] Ploi compares the current full configuration and updates it only when changed
- [ ] Forge does not claim or invoke a removed NGINX mutation endpoint
- [ ] Exceptions are caught and returned as `OperationResult::fail()`

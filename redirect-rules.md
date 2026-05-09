# Spec: Redirect Rules

**Issue:** MAR-31
**Date:** 2026-04-23
**Status:** In Review

## Overview

Redirect rules define URL path redirections from a source path to a destination path with a specified redirect type (redirect/permanent/temporary). The `RedirectManagerInterface` orchestrates creation of redirects via provider SDKs, consuming `RedirectConfig` value objects from the project.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Config/RedirectConfig.php` | Value object with from, to, type |
| `app/Deployment/Contracts/RedirectManagerInterface.php` | Interface defining `plan()` and `apply()` |
| `app/Deployment/Providers/Ploi/PloiRedirectManager.php` | Ploi implementation |
| `app/Deployment/Providers/Forge/ForgeRedirectManager.php` | Forge implementation |
| `tests/Unit/Config/RedirectConfigTest.php` | Unit tests |

### RedirectConfig

```php
final class RedirectConfig
{
    public function __construct(
        private readonly string $from,
        private readonly string $to,
        private readonly string $type = 'redirect',
    ) {}

    public function from(): string { return $this->from; }
    public function to(): string { return $this->to; }
    public function type(): string { return $this->type; }
}
```

### RedirectManagerInterface

```php
interface RedirectManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;

    /**
     * @param array<string, RedirectConfig> $redirects
     */
    public function apply(SiteContext $site, array $redirects): OperationResult;
}
```

## Functional Requirements

**FR-001 — Plan iterates all project redirects**
`plan()` iterates `$context->project->redirects()` and generates a summary string per redirect: `"Create redirect: FROM → TO (TYPE)"`.

**FR-002 — Apply creates redirects via provider API**
- Ploi: `$server->sites($siteId)->redirects()->create($from, $to, $type)`
- Forge: `$forge->createRedirectRule($serverId, $siteId, ['from' => $from, 'to' => $to, 'type' => $type])`

**FR-003 — Redirect name used as array key**
The `apply()` method receives `array<string, RedirectConfig>` — the string key is used in error messages: `"Failed to create redirect {$redirectName}: ..."`.

**FR-004 — Apply stops on first failure**
If a redirect creation throws an exception, `apply()` returns `OperationResult::fail()` immediately.

## Configuration Interface

```yaml
redirects:
  Legacy API:
    from: /api/v1
    to: /api/v2
    type: permanent
  Old Landing:
    from: /landing
    to: /
    type: redirect
```

## Data Contracts

```php
// Config object
RedirectConfig { from: string, to: string, type: string }

// Interface signatures
plan(DeploymentContext $context): array<string>
apply(SiteContext $site, array<string, RedirectConfig> $redirects): OperationResult

// Default type
RedirectConfig::type() default: 'redirect'
```

## Edge Cases

- **Empty redirects:** `plan()` returns `[]` (iterating an empty array)
- **Invalid redirect type:** Not validated at Shipper level; provider may reject
- **Self-redirect (from == to):** Not validated at Shipper level

## Acceptance Criteria

- [ ] `RedirectConfig` has correct default: `type='redirect'`
- [ ] `plan()` generates one summary string per redirect
- [ ] `apply()` calls the correct provider API method for each redirect
- [ ] Exceptions are caught and returned as `OperationResult::fail()` with redirect name
- [ ] Both providers are symmetric (same interface, same behavior)
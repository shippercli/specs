# Spec: SSL Management

**Issue:** MAR-27
**Date:** 2026-04-23
**Status:** In Review

## Overview

SSL certificate management allows projects to enable HTTPS via Let's Encrypt or custom certificate types, orchestrated through a provider-agnostic `SslManagerInterface`. Each deployment provider (Ploi, Forge) implements this interface to create and manage SSL certificates on the target platform.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Config/SslConfig.php` | Value object holding `enabled` (bool) and `type` (string) |
| `app/Deployment/Contracts/SslManagerInterface.php` | Interface defining `plan()` and `apply()` |
| `app/Deployment/Providers/Ploi/PloiSslManager.php` | Ploi implementation using `$server->sites()->certificates()->create()` |
| `app/Deployment/Providers/Forge/ForgeSslManager.php` | Forge implementation using `obtainLetsEncryptCertificate()` |
| `tests/Unit/Config/SslConfigTest.php` | Unit tests for SslConfig |

### SslConfig

```php
final class SslConfig
{
    public function __construct(
        private readonly bool $enabled = false,
        private readonly string $type = 'letsencrypt',
    ) {}

    public function enabled(): bool { return $this->enabled; }
    public function type(): string { return $this->type; }
}
```

### SslManagerInterface

```php
interface SslManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;

    public function apply(SiteContext $site, SslConfig $ssl): OperationResult;
}
```

## Functional Requirements

**FR-001 — Disabled SSL is a no-op**
When `ssl.enabled()` is `false`, `plan()` returns `[]` and `apply()` returns `OperationResult::ok()` immediately.

**FR-002 — Plan generates a human-readable summary**
`plan()` returns an array containing `"Create SSL certificate (TYPE) for domain: DOMAIN"` when SSL is enabled.

**FR-003 — Domain resolution from profile**
The domain is read from `$context->profile->get('domain')` with a fallback to empty string when the value is not a string.

**FR-004 — Apply creates certificate via provider API**
`apply()` delegates to the provider's SDK:
- Ploi: `$server->sites($siteId)->certificates()->create($siteDomain, $sslType)`
- Forge: `obtainLetsEncryptCertificate($serverId, $siteId, ['domains' => [$siteDomain]])`

**FR-005 — Apply failures return OperationResult::fail**
Any exception during apply is caught and returns `OperationResult::fail()` with the exception message.

## Configuration Interface

```yaml
ssl:
  enabled: true
  type: letsencrypt   # or: custom, wildcard
```

## Data Contracts

```php
// Config object
SslConfig { enabled: bool, type: string }

// Interface signatures
plan(DeploymentContext $context): array<string>
apply(SiteContext $site, SslConfig $ssl): OperationResult
```

## Edge Cases

- **Missing domain in profile:** `plan()` uses empty string `""` — results in "for domain:" with no domain name
- **Invalid certificate type:** No validation at config level; provider API may reject unknown types
- **Certificate already exists:** Provider SDKs may throw; error is captured in OperationResult

## Acceptance Criteria

- [ ] `SslConfig` defaults: `enabled=false`, `type='letsencrypt'`
- [ ] `plan()` returns `[]` when SSL is disabled
- [ ] `plan()` returns descriptive string when SSL is enabled
- [ ] `apply()` calls correct Ploi/Forge SDK methods with site domain and SSL type
- [ ] Exceptions during apply are caught and returned as `OperationResult::fail()`
- [ ] Both providers implement the same interface
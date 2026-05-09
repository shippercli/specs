# Spec: Environment Variables

**Issue:** MAR-23
**Date:** 2026-04-24
**Status:** In Review

## Overview
Shipper manages environment variables by merging project and profile-level configs into the existing `.env` file on the server. Variables from Shipper override existing values; new variables are appended.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Config/EnvironmentConfig.php` | Value object holding variable map |
| `app/Deployment/Contracts/EnvironmentManagerInterface.php` | Interface for providers |
| `app/Deployment/Concerns/MergesEnvironment.php` | Trait handling merge logic |
| `app/Deployment/Providers/Ploi/PloiEnvironmentManager.php` | Ploi implementation |
| `app/Deployment/Providers/Forge/ForgeEnvironmentManager.php` | Forge implementation |
| `app/tests/Unit/Config/EnvironmentConfigTest.php` | Unit tests |

## Functional Requirements

**FR-001 — Environment Merge**
Project environment variables merge with profile environment variables. Profile values override project values for the same key.

**FR-002 — File Merge Strategy**
Merge into existing `.env` content: existing keys are updated, new keys appended, comments and blank lines preserved.

**FR-003 — Apply**
On `shipper apply`, read existing `.env` from server, merge Shipper variables, write back.

**FR-004 — Plan**
`plan()` returns count of variables to be set, e.g. "Set 3 environment variables".

## Configuration Interface

```yaml
environment:
  APP_ENV: production
  APP_DEBUG: "false"
  DB_CONNECTION: mysql
```

## Data Contracts

```php
// app/Config/EnvironmentConfig.php
final class EnvironmentConfig
{
    /**
     * @param array<string, string> $variables
     */
    public function __construct(private readonly array $variables = []) {}
    public function variables(): array;
    public function isEmpty(): bool;
    public function mergeWith(self $other): self;
}

// app/Deployment/Contracts/EnvironmentManagerInterface.php
interface EnvironmentManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;
    public function apply(SiteContext $site, EnvironmentConfig $environment): OperationResult;
}

// MergesEnvironment trait
private function mergeEnvContent(string $existing, array $variables): string
```

## Edge Cases

- **Empty environment:** `isEmpty()` returns true when no variables configured; `apply()` is no-op
- **Override existing:** If `.env` already has `APP_KEY`, Shipper value replaces it
- **New key:** Appended after last non-empty, non-comment line
- **Comment lines:** Preserved exactly as-is
- **Empty lines:** Preserved, or added between sections when appending new vars

## Acceptance Criteria

- [ ] `EnvironmentConfig` stores variables as `array<string, string>`
- [ ] `mergeWith()` returns new instance, does not mutate original
- [ ] Profile environment overrides project environment for same keys
- [ ] `apply()` merges into existing `.env` content on server
- [ ] `apply()` skips operation if environment is empty
- [ ] Comments and blank lines in `.env` are preserved
- [ ] New variables are appended to end of `.env`
- [ ] `plan()` returns descriptive message with variable count

## Open Questions / Potential Concerns

- No mechanism to delete environment variables (only add/override)
- No support for multiline values or special characters in variable values
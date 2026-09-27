---
description: "**Issue:** MAR-20 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Provider Architecture

**Issue:** MAR-20  
**Date:** 2026-04-23  
**Status:** In Review

## Overview

Shipper's deployment providers are Composer plugins. The CLI owns the
configuration and deployment entry point; provider packages own SDK-specific
operations and capability declarations. Providers are not a monolithic set of
manager classes inside the CLI.

## Current implementation

| Component | Responsibility |
|---|---|
| `shippercli/contracts` | Shared deployment, logs, status, and capability contracts |
| `shippercli/cli` | Loads configuration, discovers `shipper-plugin` packages, and invokes providers |
| `shippercli/provider-ploi` | Ploi API integration and Ploi-specific capability implementations |
| `shippercli/provider-forge` | Forge API v2 integration with explicit partial capabilities |
| `shippercli/provider-easypanel` | EasyPanel API integration and ownership-safe service cleanup |
| `shippercli/provider-cpanel` | cPanel UAPI/API2 integration with manifest-backed cleanup |

Provider discovery uses Composer package type `shipper-plugin` and the package
metadata key `extra.shipper-plugin`. The CLI must not contain provider SDK
dependencies or dead in-tree provider implementations.

## Provider contract summary

Every provider implements `DeploymentProviderInterface`:

```php
interface DeploymentProviderInterface
{
    public function getName(): string;
    public function validate(object $project, object $profile): array;
    public function plan(object $project, object $profile): array;
    public function apply(object $project, object $profile): bool;
    public function destroy(object $project, object $profile): bool;
    public function getLastError(): string;
}
```

Optional contracts expose logs, deployment status, and the shared capability
manifest. Provider-specific operations remain behind the provider package's
client boundary and are not represented as obsolete CLI manager interfaces.

## Apply and destroy behavior

The common high-level sequence is:

1. Validate provider configuration and project/profile values.
2. Resolve or create the provider deployment target.
3. Reconcile resources that the provider supports.
4. Trigger deployment and expose status/logs where supported.
5. Destroy only resources whose provider-specific ownership rules are proven.

The sequence is capability-dependent. A provider must reject unsupported
configuration rather than silently ignore it. It may report a capability as
`partial` when the provider API supports only a documented subset.

## Capability differences

Provider implementations are intentionally not symmetric. For example:

- Ploi supports redirects, network rules, PHP versions, and raw NGINX
  configuration with mocked and CI-backed coverage.
- Forge API v2 supports site deployment, domains, certificates, databases,
  environment, workloads, logs, and ownership-gated preview-site cleanup; it
  does not claim server lifecycle or the removed Forge v1 mutation operations.
- EasyPanel uses environment ownership markers and supports core orphan-preview
  cleanup for marked app services.
- cPanel uses an account manifest and requires ownership evidence before
  destroying domains, databases, or account resources.

The canonical capability state is the provider's `capabilities()` manifest and
the corresponding provider documentation, not an assumption that every
provider implements every operation.

## Safety requirements

- Provider packages must be discovered by Composer plugin metadata.
- Unsupported operations fail validation with a provider-specific message.
- Existing resources are never adopted or modified without the provider's
  ownership evidence.
- Destroy operations are idempotent for missing owned resources and fail closed
  for unmanaged resources.
- Provider API exceptions must become actionable `getLastError()` messages.
- Provider documentation must identify API-version limitations and cleanup
  limitations explicitly.

## Acceptance criteria

- [ ] CLI provider discovery loads `shipper-plugin` packages without provider
      SDKs in the application package.
- [ ] Each provider publishes a manifest conforming to the shared capability
      contract.
- [ ] Unsupported provider mutations are rejected or reported as unsupported.
- [ ] Ownership checks protect apply reconciliation and destroy operations.
- [ ] Provider-specific tests cover supported operations and safety failures.
- [ ] Documentation and website mirrors describe current API versions and
      capability differences.

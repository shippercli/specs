---
description: "**Issue:** MAR-34 **Date:** 2026-04-24 **Status:** In Review"
---

# Spec: Repository Management

**Issue:** MAR-34  
**Date:** 2026-04-24  
**Status:** In Review

## Current contract

Repository setup is provider-specific. Ploi can install a repository through
its API. Forge API v2, used by `shippercli/provider-forge`, removed Git
repository mutation endpoints; a Forge site must therefore have its source
configured in Forge before Shipper triggers deployment. Shipper must not claim
that it can install a Forge repository or pass the removed v1 `composer`
payload.

## Provider behavior

| Provider | Repository setup | Deployment behavior |
|---|---|---|
| Ploi | Installs provider/repository/branch through the Ploi API | Installs and deploys |
| Forge | Requires a preconfigured Forge site source | Resolves the site and triggers API v2 deployment |
| cPanel | Uses the provider's configured deployment mode | Provider-specific |
| EasyPanel | Configures the declared source through the EasyPanel API | Deploys the app service |

## Requirements

**FR-001 — Provider-specific source handling**  
Each provider must either configure the source through a supported API or
return a clear limitation before mutation.

**FR-002 — Branch handling**  
Providers that expose branch configuration must use the profile branch. A
provider without a repository mutation endpoint must not silently ignore it.

**FR-003 — Plan description**  
`plan()` must describe whether the source will be installed, reused, or must
be preconfigured by the provider.

**FR-004 — Failure propagation**  
Unsupported source mutation returns an actionable provider error and never
reports a successful deployment.

## Forge API v2 limitation

The official Forge SDK v4 requires an organization slug and no longer exposes
the v1 Git repository mutation methods. The Forge provider therefore requires
`api_token`, `organization_slug`, and `server_id`; it creates/resolves the
site and triggers deployment, but source setup remains a Forge-side
precondition.

## Acceptance criteria

- [ ] Ploi source installation remains covered by provider tests.
- [ ] Forge never calls a removed v1 Git endpoint.
- [ ] Forge plans and documentation state the preconfigured-source limitation.
- [ ] Unsupported source mutation fails clearly instead of succeeding.
- [ ] Provider-specific repository behavior is represented in the capability
      matrix and website mirror.

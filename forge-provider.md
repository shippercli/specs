---
description: "**Issue:** MAR-39 **Date:** 2026-04-23 **Status:** In Review"
---

# Spec: Forge Provider

**Issue:** MAR-39  
**Date:** 2026-04-23  
**Status:** In Review

## Current implementation

`shippercli/provider-forge` is a Composer plugin built around the official
`laravel/forge-sdk` v4 API v2 client. It is intentionally partial: it resolves
or creates a site, triggers deployment, and protects cleanup with ownership
tags. It can also create or reuse a server from an explicit lifecycle map and
optionally clean up only servers carrying its ownership tag. It provisions
configured databases, environment variables, queue workers, scheduled jobs,
certificates, and application logs through the Forge API v2 client. Deployment
rollback remains outside this provider contract.

| Component | Responsibility |
|---|---|
| `ForgeProvider` | Validate configuration, plan, apply, and destroy |
| `ForgeClientInterface` | Injectable provider operation boundary |
| `ForgeApiClient` | Forge SDK v4/API v2 adapter |
| `ForgeProviderTest` | Capability, deployment, and ownership regression tests |

## Configuration

The provider requires:

- `api_token`
- `organization_slug`
- either `server_id` or a `server` lifecycle map

The lifecycle map must include `provider`, `credential_id`, `name`, `type`,
`size`, and `region`. Set `cleanup: true` only when Shipper should remove the
owned server during destroy.

The optional `ownership_tag` defaults to `shipper-managed`.

## Supported behavior

1. Find a site by its configured domain on the selected server.
2. Create a PHP site when no matching site exists.
3. Trigger deployment through the Forge API v2 SDK.
4. Refuse destruction unless the returned site contains the ownership tag.
5. Apply configured databases, environment variables, workers, scheduled jobs,
   and certificates before deployment.

Forge API v2 removed Git repository mutation endpoints. The site source must
be configured in Forge before Shipper triggers deployment; the provider must
not emulate the removed API v1 operation.

## Capability state

| Capability | State | Limitation |
|---|---|---|
| App deployment | Partial | Source setup is a Forge-side prerequisite. |
| Domain management | Supported | Site domain resolution/creation is supported. |
| SSL, databases, environment, PHP, NGINX | Supported | Resources are applied through Forge API v2. |
| Workers, cron, observability | Supported | Existing resources are reused by name or command. |
| Rollback | Unsupported | Forge API v2 exposes deployment history but no safe rollback mutation. |
| Server lifecycle | Supported | Creation/reuse and opt-in cleanup are ownership-tagged. |
| Previews | Unsupported | No ownership-safe implementation yet. |

## Safety requirements

- API v2 credentials and organization scope must be explicit.
- A missing site ID or API exception fails the operation.
- Destroy is idempotent for a missing site.
- Destroy refuses an untagged site.
- No provider operation may report success without an observable Forge API
  operation.

---
description: "EasyPanel provider capability contract and ownership rules."
---

# Spec: EasyPanel Provider

**Date:** 2026-10-16
**Status:** In Review

## Current implementation

`shippercli/provider-easypanel` uses the EasyPanel API to manage an owned
project, application service, databases, queue workers, and cron scheduler.
Queue workers are represented as dedicated App services; cron entries are
represented as scripts in a managed Box service. Daemon/process supervision,
resource limits, mounts, and persistent-volume lifecycle are not implemented by
the provider.

## Capability state

| Capability | State | Limitation |
|---|---|---|
| App deployment | Supported | Source and deployment settings use EasyPanel API procedures. |
| Databases | Supported | Newly created database services are ownership-marked and cleaned up safely; existing unmarked databases cannot be adopted because the API client has no database environment update operation. |
| Queue workers | Partial | Workers run as separately managed App services; daemon/process supervision is not implemented. |
| Cron | Supported | Requires a Git-backed source and uses a managed Box scheduler. |
| Resource limits, mounts, persistent volumes | Unsupported | These API operations are not implemented by the provider. |
| Logs | Partial | Requires EasyPanel log aggregation. |
| Rollback | Unsupported | No safe public rollback mutation is exposed. |
| Previews | Partial | Owned preview app services can be enumerated and removed through the core orphan-cleanup flow; EasyPanel domain cleanup remains provider API dependent. |

## Safety requirements

- Workload services must carry the Shipper ownership marker and project marker.
- Destroy must refuse to remove a workload without its matching ownership marker.
- Unrelated EasyPanel services must never be removed.
- Adopted or unmarked databases must be preserved; the current API client cannot safely add ownership markers to an existing database service.

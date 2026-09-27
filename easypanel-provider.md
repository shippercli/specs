---
description: "EasyPanel provider capability contract and ownership rules."
---

# Spec: EasyPanel Provider

## Current implementation

`shippercli/provider-easypanel` uses the EasyPanel API to manage an owned
project, application service, databases, queue workers, cron scheduler, and
daemon services. Queue workers and daemons are represented as dedicated App
services; cron entries are represented as scripts in a managed Box service.

## Capability state

| Capability | State | Limitation |
|---|---|---|
| App deployment | Supported | Source and deployment settings use EasyPanel API procedures. |
| Databases | Supported | Managed database services are ownership-marked and cleaned up safely. |
| Queue workers and daemons | Supported | Each workload is a separately managed App service. |
| Cron | Supported | Requires a Git-backed source and uses a managed Box scheduler. |
| Logs | Partial | Requires EasyPanel log aggregation. |
| Rollback | Unsupported | No safe public rollback mutation is exposed. |
| Previews | Partial | Automated preview cleanup is not implemented. |

## Safety requirements

- Workload services must carry the Shipper ownership marker and project marker.
- Destroy must refuse to remove a workload without its matching ownership marker.
- Unrelated EasyPanel services must never be removed.

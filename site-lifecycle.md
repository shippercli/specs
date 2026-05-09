# Spec: Site Lifecycle Management

**Issue:** MAR-21
**Date:** 2026-04-24
**Status:** In Review

## Overview

Shipper manages the complete lifecycle of sites on deployment providers (Ploi): creating, updating, deploying, and destroying sites. The lifecycle is driven by two primary flows — `ApplyDeploymentFlow` for create/update operations and `DestroyDeploymentFlow` for site teardown.

## Current Implementation

### Key Classes / Files

| Class/File | Type | Responsibility |
|---|---|---|
| `ApplyDeploymentFlow` | Flow | Orchestrates plan + execute for deployments |
| `DestroyDeploymentFlow` | Flow | Orchestrates plan + execute for site destruction |
| `CreateDeploymentPlanAction` | Action | Delegates to provider to build deployment plan |
| `ExecuteDeploymentAction` | Action | Calls `provider->apply()` to trigger deployment |
| `DestroySiteAction` | Action | Calls `provider->destroy()` to remove a site |
| `DeleteSiteAction` | Action | Deletes a site by ID via `SiteManagerInterface` |
| `GetAllSitesAction` | Action | Lists all sites from provider via `ServerSiteListInterface` |
| `ValidateProjectAction` | Action | Validates project + profile configuration via provider |
| `SiteManagerInterface` | Contract | `plan()`, `apply()`, `destroy()` for site operations |
| `ServerSiteListInterface` | Contract | `getAllSites()` to enumerate provider sites |
| `RepositoryManagerInterface` | Contract | `plan()`, `apply()` for repository operations |

### Lifecycle Flow

**Apply (create/update):**
```
ApplyDeploymentFlow::handle() → LoadConfiguration → ValidateProject → CreateDeploymentPlan
                                           ↓
                            ApplyDeploymentFlow::execute() → ExecuteDeploymentAction → provider.apply()
```

**Destroy (teardown):**
```
DestroyDeploymentFlow::handle() → LoadConfiguration → ValidateProject → CreateDeploymentPlan
                                           ↓
                        DestroyDeploymentFlow::execute() → DestroySiteAction → provider.destroy()
```

## Functional Requirements

**FR-001 — Site Creation**
When `shipper apply` runs for a new site (domain not found on server), Shipper creates the site on the provider with repository configuration, web directory settings, and triggers an initial deployment.

**FR-002 — Site Updates**
When `shipper apply` runs for an existing site (domain found), Shipper updates configuration and triggers a new deployment from the configured branch.

**FR-003 — Site Discovery**
Sites are identified by domain name, not by ID. The `GetAllSitesAction` returns `array<int, array{site_id: int, domain: string}>` by querying `ServerSiteListInterface`.

**FR-004 — Site Destruction**
When `shipper destroy` runs, all linked databases are deleted, database users are removed, and the site itself is destroyed via `SiteManagerInterface::destroy()`.

**FR-005 — Deployment Planning**
`CreateDeploymentPlanAction` delegates to the provider's `plan()` method, returning `array<string, mixed>` with server_id, site_id, and other deployment metadata.

**FR-006 — Validation Before Deployment**
`ValidateProjectAction` calls `$provider->validate($project, $profile)` before any plan or apply operation. Returns `array<int, string>` of error messages.

## Configuration Interface / Command Interface

```bash
# Plan (dry-run)
./shipper plan <project> --profile=<profile>

# Apply (create or update)
./shipper apply <project> --profile=<profile> [--force]

# Destroy (teardown)
./shipper destroy <project> --profile=<profile> [--force]

# Validate configuration
./shipper validate
```

### Profile Configuration (shipper.yml)

```yaml
projects:
  api:
    provider: ploi
    repository:
      provider: github
      name: owner/repo
    web_directory: /public
    project_root: /
    profiles:
      production:
        domain: api.example.com
        branch: main
      staging:
        domain: api-staging.example.com
        branch: develop
      preview:
        domain: "api-preview-${GITHUB_PR_NUMBER}.example.com"
        branch: "${GITHUB_HEAD_REF}"
```

## Data Contracts

```php
// SiteManagerInterface
interface SiteManagerInterface {
    /** @return array<string> */
    public function plan(DeploymentContext $context): array;
    public function apply(DeploymentContext $context): SiteResult;
    public function destroy(SiteContext $site): OperationResult;
}

// ServerSiteListInterface
interface ServerSiteListInterface {
    /** @return array<int, array{site_id: int, domain: string}> */
    public function getAllSites(ServerContext $server): array;
}

// RepositoryManagerInterface
interface RepositoryManagerInterface {
    /** @return array<string> */
    public function plan(DeploymentContext $context): array;
    public function apply(SiteContext $site, DeploymentContext $context): OperationResult;
}
```

## Edge Cases

- **Domain already exists on provider:** `plan()` detects this; apply flow skips creation and updates existing site
- **Invalid server ID:** `GetAllSitesAction` throws `InvalidArgumentException` if server ID is non-numeric
- **Site manager not available:** `DeleteSiteAction` throws `RuntimeException` if provider doesn't expose `SiteManagerInterface`
- **Validation failures:** Flow returns early with `errors` array and `error_message`, never proceeds to plan/apply
- **PR preview cleanup gap:** If PR is merged without closing, the `cleanup-preview.yml` workflow does not fire (only triggers on `closed` event)

## Acceptance Criteria

- [ ] `shipper plan` returns structured plan with server_id, site_id, domain, branch
- [ ] `shipper apply` creates site if domain not found, updates if found
- [ ] `shipper destroy` removes site and all linked databases
- [ ] `GetAllSitesAction` returns sites as `array<int, array{site_id: int, domain: string}>`
- [ ] `DeleteSiteAction` requires numeric server ID, throws on invalid
- [ ] Both flows (`ApplyDeploymentFlow`, `DestroyDeploymentFlow`) support injectable provider resolver for testing
- [ ] Validation errors prevent plan/apply execution and return structured error array

## Open Questions / Potential Concerns

- **Orphaned previews on unmerged PRs:** The cleanup workflow only triggers on `pull_request: closed`. If a PR is force-pushed to main without closing, preview sites accumulate. MAR-37 addresses this via the orphan cleanup command.
- **Deployment timeout:** The docs mention a `deployment_timeout` config but no corresponding implementation in the action/flow classes read. Is this wired up?
- **Zero-downtime deploys:** Referenced in docs but not implemented in the flow classes. Any intention to implement rolling deploys?
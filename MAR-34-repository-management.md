# Spec: Repository Management

**Issue:** MAR-34
**Date:** 2026-04-24
**Status:** In Review

## Overview
Shipper installs a Git repository on a deployed site by calling the provider's API with provider name, repository name, and branch. Both Ploi and Forge are supported.

## Current Implementation

### Key Classes / Files

| File | Role |
|------|------|
| `app/Deployment/Contracts/RepositoryManagerInterface.php` | Interface defining plan/apply |
| `app/Deployment/Providers/Ploi/PloiRepositoryManager.php` | Ploi implementation |
| `app/Deployment/Providers/Forge/ForgeRepositoryManager.php` | Forge implementation |

## Functional Requirements

**FR-001 — Repository Install**
On `shipper apply`, install the configured Git repository onto the site using the provider's API.

**FR-002 — Branch from Profile**
The branch to deploy comes from `$context->profile->branch()`.

**FR-003 — Plan Description**
`plan()` returns "Install repository: {provider}:{name} ({branch})".

**FR-004 — Composer Install**
Forge provider passes `composer: true` in the API payload to trigger Composer install post-deployment.

## Data Contracts

```php
// app/Deployment/Contracts/RepositoryManagerInterface.php
interface RepositoryManagerInterface
{
    /**
     * @return array<string>
     */
    public function plan(DeploymentContext $context): array;
    public function apply(SiteContext $site, DeploymentContext $context): OperationResult;
}

// Repository data shape (from ProjectConfig):
[
    'provider' => 'github',  // or 'gitlab', 'bitbucket'
    'name' => 'organization/repository',
]

// Profile branch:
// $context->profile->branch() → string (e.g., "main", "production")
```

## Edge Cases

- **Missing provider/name:** `plan()` returns "Install repository: unknown:unknown" if not set
- **Install failure:** `apply()` returns `OperationResult::fail()` with exception message
- **Provider-specific payload differences:** Ploi and Forge have different API structures

## Acceptance Criteria

- [ ] `RepositoryManagerInterface` has `plan(DeploymentContext): array<string>` and `apply(SiteContext, DeploymentContext): OperationResult`
- [ ] Ploi provider calls `$siteObj->repository()->install($provider, $branch, $repoName)`
- [ ] Forge provider calls `$forge->installGitRepositoryOnSite($serverId, $siteId, [provider, repository, branch, composer])`
- [ ] `plan()` includes provider, repository name, and branch in description
- [ ] Failure during apply returns `OperationResult::fail()` with message

## Open Questions / Potential Concerns

- No repository uninstall/destroy operation defined (future feature)
- No branch protection or deployment controls
- No way to override composer behavior per-environment
- Ploi and Forge differ in API structure — no shared abstraction beyond the interface
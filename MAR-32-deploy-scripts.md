# Spec: Deploy Scripts

**Issue:** MAR-32
**Date:** 2026-04-24
**Status:** In Review

## Overview

Deploy scripts define the bash commands that run on the server during a deployment. Shipper resolves a script from project config (and optionally overridden by profile config), performs placeholder interpolation, and uses a provider-specific manager to push the resolved script to the hosting provider.

## Current Implementation

### Key Classes / Files

| Class/File | Role |
|---|---|
| `App\Deployment\Concerns\ResolvesDeployScript` | Trait with `resolveDeployScript()` logic |
| `App\Deployment\Contracts\DeployScriptManagerInterface` | Interface defining `plan()` and `apply()` |
| `App\Deployment\Providers\Ploi\PloiDeployScriptManager` | Updates deploy script via Ploi API |
| `App\Deployment\Providers\Forge\ForgeDeployScriptManager` | Updates deploy script via Forge API |

## Functional Requirements

**FR-001 — Script Resolution Priority**
Profile-level `deploy_script` completely overrides project-level script. If neither is set or both are empty strings, no deploy script is configured.

**FR-002 — Placeholder Interpolation**
Resolved scripts undergo two substitutions: `{site}` is replaced with the profile's `domain`, and `{branch}` is replaced with the profile's `branch`.

**FR-003 — Plan Output**
`plan()` returns `['Update deployment script']` when a script is present, or an empty array when no script is configured.

**FR-004 — Apply via Provider**
`apply()` resolves the script, then calls the provider API to set or update the deployment script. If resolved script is empty, `apply()` returns `OperationResult::ok()` with no API call.

**FR-005 — Per-Site Interpolation**
The `{site}` placeholder uses the domain from the specific site being deployed, not a global value.

## Configuration Interface

```yaml
# Project level (shipper.php)
deploy_script: |
  cd /home/ploi/{site}
  git pull origin {branch}
  composer install --no-dev --optimize-autoloader
  php artisan migrate --force
  php artisan queue:restart

# Profile level (overrides project level)
profiles:
  production:
    domain: example.com
    branch: main
    deploy_script: |
      cd /home/ploi/{site}
      git pull origin {branch}
      composer install --no-dev
      php artisan migrate --force
```

## Data Contracts

```php
// ResolvesDeployScript (trait)
trait ResolvesDeployScript {
    private function resolveDeployScript(ProjectConfig $project, ProfileConfig $profile): string {
        // Profile deploy_script takes priority over project-level
        // {site} → domain from profile
        // {branch} → branch from profile
        // Returns '' if both are empty
    }
}

// DeployScriptManagerInterface
interface DeployScriptManagerInterface {
    /** @return array<string> */
    public function plan(DeploymentContext $context): array;

    public function apply(SiteContext $site, DeploymentContext $context): OperationResult;
}
```

## Edge Cases

- **Empty script at both levels:** `plan()` returns `[]`, `apply()` returns `OperationResult::ok()` without calling API.
- **`{site}` not present in script:** Interpolation is a no-op for that placeholder.
- **`{branch}` not present in script:** Interpolation is a no-op for that placeholder.
- **Provider API throws:** Returns `OperationResult::fail()` with descriptive error.
- **Whitespace-only script:** Treated as non-empty string; passed to provider.

## Acceptance Criteria

- [ ] Profile-level `deploy_script` overrides project-level when present
- [ ] `{site}` is replaced with the site domain, `{branch}` with the profile branch
- [ ] When both project and profile have empty scripts, `plan()` returns `[]` and `apply()` succeeds without API call
- [ ] Ploi provider calls `server->sites(...)->deployment()->updateDeployScript()`
- [ ] Forge provider calls `forge->updateSiteDeploymentScript()`
- [ ] Provider exceptions result in `OperationResult::fail()`

## Open Questions / Potential Concerns

- Should Shipper validate the script structure (e.g., require certain commands like `composer install`)?
- Should scripts be stored somewhere (e.g., a file path reference) instead of inline in the config?
- Is there a maximum script length that providers accept?
